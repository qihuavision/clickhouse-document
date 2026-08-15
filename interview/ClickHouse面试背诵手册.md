# 🎯 ClickHouse 面试背诵手册（划重点版）

> 使用方法：**加粗黑体 = 关键词必须说出口**，<mark>黄色高亮 = 加分句/数字</mark>。
> 每段就是一道题的完整回答，照着背，背熟直接输出。

---

# 第一部分：MergeTree 选型背诵卡（5 张）

## 卡 1：日志、埋点、事件流 → 普通 MergeTree

**【背诵】**
"日志埋点这类数据<mark>写完就不会改</mark>，直接用**普通 MergeTree** 就够了，生产上 70% 的场景都是它。<mark>不需要更新就绝不用变体引擎</mark>，因为变体在后台合并时要多干活，白白损失写入吞吐。"

**【追问】** 那数据重复了怎么办？→ "业务层保证幂等，或靠 ClickHouse 的数据块去重机制，不需要引擎层去重。"

---

## 卡 2：订单状态、用户资料、商品价格 → ReplacingMergeTree

**【背诵】**
"订单、用户资料这类数据是<mark>同一主键多次写入、以最新为准</mark>，选 **ReplacingMergeTree**，建表时指定版本列 `ReplacingMergeTree(version)`，合并时自动按主键留版本最新的一条。**关键点是查询必须配合**：要么用 `argMax(status, version) GROUP BY order_id` 取最新值，要么加 FINAL。因为<mark>去重发生在后台合并，是异步的</mark>，直接 SELECT 会查出重复行和旧状态，这是设计行为不是 bug。FINAL 能用但慢，<mark>大表上比 argMax 慢 10 到 50 倍</mark>，只留给低频对账。"

**【追问】** 为什么 CH 不做写入时去重？→ "写入时去重要查已有数据，毁掉批量写入的高吞吐，CH 用异步合并换写入性能。"

---

## 卡 3：实时大屏 GMV/UV、多维报表 → AggregatingMergeTree + 物化视图

**【背诵】**
"大屏和报表的场景是<mark>明细数据巨大但只要聚合结果</mark>，标准方案是 **AggregatingMergeTree 加物化视图**做预聚合：物化视图在明细写入时自动增量聚合，用 `uniqState`、`sumState` 存聚合状态，目标表用 AggregatingMergeTree，合并时状态自动融合；查询时用 `uniqMerge`、`sumMerge` 收口。**效果是亿级明细、毫秒级查询**。<mark>UV 这种去重计数必须用 uniqState/uniqMerge</mark>，因为它底层是 HyperLogLog，状态可合并，能跨分片算；直接 count distinct 会把数据全拉到一台机器上算，内存爆炸。"

**【追问】** 物化视图会回填历史吗？→ "不会，只对新插入触发；POPULATE 有竞争风险，历史数据要手动 INSERT SELECT 补。"

---

## 卡 4：分钟级指标、金额累计 → SummingMergeTree

**【背诵】**
"只要**加法汇总**的场景用 **SummingMergeTree**，比如分钟级监控指标、金额累计。它合并时对<mark>数值列自动 SUM</mark>。两个必须记住的坑：一是<mark>查询时必须再 sum 一次</mark>，因为不同 part 可能还有同键的部分和；二是<mark>非数值列不会被聚合，合并时取任意行</mark>，所以字符串维度列必须放进 ORDER BY，否则查出随机值。需要 uniq、分位数这种复杂聚合时就不能用它，要换 AggregatingMergeTree。"

---

## 卡 5：购物车、会话状态流转 → VersionedCollapsingMergeTree

**【背诵】**
"状态频繁流转、又要保留变更历史的场景，比如购物车、会话，用 **VersionedCollapsingMergeTree**。原理是把更新变成两条插入：<mark>一条 sign=-1 的行抵消旧状态，一条 sign=+1 的新行</mark>，合并时正负抵消。查询要用 `sum(sign)` 聚合。**生产上直接用它，别用裸的 CollapsingMergeTree**，因为裸的<mark>要求同一会话的写入严格有序</mark>，多线程多客户端乱序写就抵消失败、数据膨胀；Versioned 靠版本号配对，不怕乱序。"

---

## 选型一句话总结（背这段）

> "**不更新用普通 MergeTree；要最新版本用 Replacing 配 argMax；只要 SUM 用 Summing；要 UV 等复杂聚合用 Aggregating 加物化视图；状态机流转用 VersionedCollapsing。所有变体的特殊逻辑都发生在后台合并时，异步最终生效——所以查询方式必须配合引擎，用错查询方式等于又慢又错。**"

---

# 第二部分：高频面试题背诵版（20 题）

## Q1 列存为什么比行存快？

"ClickHouse 快是**四板斧协同**：**第一列存**，同列数据紧凑连续，只读需要的列；**第二向量化执行**，一次处理一批 8192 行而不是一行，<mark>虚函数调用摊薄上千倍</mark>；**第三 SIMD**，一条 AVX-512 指令<mark>同时处理 16 个值</mark>；**第四高压缩**，同列类型一致压缩比 5 到 10 倍，省的其实是<mark>内存带宽</mark>。底层本质是顺应 CPU 物理特性：数据连续缓存命中率高、顺序访问硬件预取生效。"

---

## Q2 稀疏索引和 MySQL 的 B+Tree 有什么区别？

"B+Tree <mark>每行一条索引</mark>，树高 3 到 4 次定位精确到行，是为点查设计的；ClickHouse 稀疏索引<mark>每 8192 行一条</mark>，8.87M 行的表索引只有 <mark>1083 条、约 100KB，全量常驻内存</mark>，二分 19 次圈定 granule，再用 SIMD 在 8192 行里过滤。本质区别：**B+Tree 优化随机 IO 次数，稀疏索引优化扫描总量**；稀疏索引为剪枝而生不为点查而生，代价是最坏多读 16384 行冗余。"

---

## Q3 granule 是什么？为什么是 8192？

"granule 是 ClickHouse <mark>读取和索引的最小单位</mark>，默认 8192 行，还有自适应粒度，单 granule 字节到 10MB 也会提前切分。8192 是权衡：太小索引和 mark 文件膨胀，太大剪枝粗糙、顺带扫太多行。**查询永远整块读 granule，不能只读一行**，这就是 CH 点查弱的根源。"

---

## Q4 排序键怎么设计？

"三条铁律：**第一列必须是过滤最频繁的列**；<mark>低基数在前、高基数在后</mark>，因为前置列基数太高会让后面的列失去单调性、退化为全表扫；时间列一般放最后，靠分区剪枝。如果两个维度查询频率相当，比如既按 user_id 又按 device_id 查，排序键只能选一个，<mark>另一个用 Projection 建第二套排序</mark>，优化器自动选择，代价是存储翻倍。"

---

## Q5 分区能加速查询吗？

"**不能，官方原话：Partitioning does not speed up queries**。<mark>加速查询靠排序键，分区服务于数据生命周期管理</mark>——按时间 TTL 删旧数据、冷热分离迁移、DROP PARTITION 快速删数据。分区一般按月，<mark>别按天分区</mark>，否则分区数爆炸、Keeper 元数据膨胀、merge 跟不上。删历史数据用 DROP PARTITION 是元数据操作，比 DELETE 快几个数量级。"

---

## Q6 为什么必须批量写入？Too many parts 是什么？

"ClickHouse <mark>每次 INSERT 都产生一个 part</mark>，后台合并速度有限。如果每秒 1000 次小写入，part 产生速度远超合并速度，单分区 active part 超过 <mark>3000 个（parts_to_throw_insert 默认值）就报 Too many parts，直接拒绝写入</mark>。而且 part 太多查询也慢，因为要枚举所有 part。**标准做法：每批至少 1 万行，每秒不超过 1 到 2 次 INSERT**，前面加 Kafka 攒批，或者用 Async Insert 让服务端攒批。"

---

## Q7 ReplacingMergeTree 查出重复数据怎么办？

"这是<mark>设计内行为</mark>：去重发生在后台 merge 时，异步最终生效，而且重复行还在不同 part 里时查询永远看得到。解决：**常规查询用 `argMax(列, version) GROUP BY 主键`**，性能好；FINAL 简单但慢只留给低频场景；OPTIMIZE FINAL 治标不治本。**生产推荐 argMax 方案**。"

---

## Q8 FINAL 为什么慢？

"FINAL 会在<mark>查询时强制应用合并逻辑</mark>，相当于运行时做一次虚拟合并，无法充分并行，<mark>part 越多越慢，大表上比 argMax 慢 10 到 50 倍</mark>，并发一高就把 CPU 打满。"

---

## Q9 Mutation 是什么？为什么要避免？

"ALTER UPDATE/DELETE 这类改数据的操作叫 Mutation，它<mark>不是原地改行，而是重写整个 part</mark>：读全部行、应用谓词、写新 part，复用 merge 机制。代价是<mark>I/O 和 CPU 极重、和 merge 抢后台线程池、大表可能跑几小时到几天、不可回滚</mark>。生产上用 ReplacingMergeTree 或 CollapsingMergeTree 代替更新，大批删除用 DROP PARTITION，小规模删除用轻量级 DELETE FROM。"

---

## Q10 JOIN 时哪张表进内存？大表 JOIN 怎么办？

"**右表建哈希表，所以小表放右边**。右表太大 OOM 时换算法：<mark>grace_hash</mark>，哈希分桶落盘逐桶处理，大表 JOIN 大表不 OOM；右表是按 JOIN 键排序的大表可以用 <mark>direct</mark>，直接稀疏索引点查几乎零内存。生产上最优解还是<mark>把维度表复制到每个分片做本地 JOIN</mark>，避免分布式 JOIN。"

---

## Q11 复制的原理？Keeper 起什么作用？

"ClickHouse 用 ReplicatedMergeTree 做异步多主复制。核心一句话：<mark>复制的不是数据，是有序的操作日志</mark>。插入、合并、mutation 记录成日志条目写到 Keeper，其他副本通过 watch 感知、拉取日志、直接从别的副本 fetch part 数据。<mark>数据本身不经过 Keeper，走副本间点对点传输</mark>。合并是确定性的，所以 Keeper 把合并任务分配给一个副本执行，其他副本直接拉结果，省 CPU。因为日志异步回放，所以<mark>副本是最终一致的</mark>。Keeper 用 Raft 多数派共识，3 节点起步，防止脑裂。"

---

## Q12 副本延迟怎么排查处理？

"副本延迟就是<mark>回放复制日志的速度跟不上写入速度</mark>，看 system.replicas 的 absolute_delay 和 queue_size。根因一般是四个：<mark>网络带宽打满、副本机器资源不足、大量大 mutation 拖慢、Keeper 抖动</mark>。对应处理：扩带宽、扩机器资源、控制 mutation 错峰、检查 Keeper 健康。读流量用 load_balancing 策略避开高延迟副本。"

---

## Q13 数据写本地表还是分布式表？GLOBAL IN 是干嘛的？

"**生产写本地表**，就是每个分片的 Replicated 表。写分布式表要多一跳转发、吞吐受单节点限制、节点宕机可能丢数据。GLOBAL IN 解决分布式查询的正确性问题：<mark>普通 IN 在分布式表上每个分片独立执行子查询，只看到本分片数据，结果错</mark>；GLOBAL IN 让发起节点先算一次子查询再广播到所有分片。但结果集太大也会有问题，最佳实践还是维度表复制到每个分片。"

---

## Q14 跨分片 count distinct 怎么算？

"必须用 <mark>uniqState 存状态、uniqMerge 合并</mark>。uniq 底层是 HyperLogLog，状态可合并，各分片算自己的状态、发起节点合并，网络只传少量状态。直接 count distinct 会把所有 uid 拉到一台机器上算，内存爆炸。"

---

## Q15 ClickHouse 物化视图和 MySQL/PG 的区别？

"PG 的物化视图是<mark>查询快照，要手动或定时 REFRESH 全量重算</mark>；ClickHouse 的物化视图本质是<mark>插入触发器</mark>，每次明细表 INSERT 自动把新数据增量聚合写入目标表，实时、自动、不重算历史。坑：只对新插入触发不回填历史；挂多个 MV 会拖慢写入，一般不超过三五个。"

---

## Q16 Projection 是什么？和物化视图怎么选？

"Projection 是<mark>同一张表内按另一套排序键存的副本</mark>，优化器自动选择，业务 SQL 不用改；物化视图是独立的表，查询要显式指定。**选型：只想换排序键加速、SQL 不想动，用 Projection；要预聚合减行数或行过滤，用 MV**。Projection 代价是存储翻倍、写入变慢，一般不超过两个。它随 part 自动维护，不需要手动刷新。"

---

## Q17 TTL 都能做什么？

"四个功能：<mark>DELETE 删数据、RECOMPRESS 重压缩、TO VOLUME 迁移存储层、DELETE 列</mark>。典型配置：30 天换 ZSTD 高压缩、90 天迁到 S3 冷盘、一年整行删除。注意 <mark>TTL 是异步的，在后台 merge 时执行</mark>，不是实时；急用可以 MATERIALIZE TTL 手动触发。"

---

## Q18 冷热分层怎么做？

"配 storage policy，多个 volume 多个 disk：<mark>热数据放本地 NVMe 用 LZ4，冷数据 TTL 迁到 S3 用 ZSTD 加 s3_cache</mark>。迁移是异步的，监控 system.parts 的 disk_name。好处是成本降 5 到 10 倍。"

---

## Q19 查询突然变慢怎么排查？

"背顺序：**第一步查 system.query_log**，对比 read_rows 和 duration；**第二步二分**：<mark>read_rows 暴涨就是剪枝失效</mark>，查 WHERE 有没有命中排序键前列、有没有函数包裹列破坏单调性，比如 `toDate(dt)=x` 要改成范围查询；<mark>read_rows 正常就是资源问题</mark>，查并发争抢、merge、mutation、内存溢写。第三步看 system.parts 的 part 数是不是暴涨，第四步 EXPLAIN 验证执行计划。"

---

## Q20 能不能用 ClickHouse 替代 MySQL？

"不能，两者负载不同。三个角度：<mark>CH 没有 ACID 事务</mark>，订单的多表一致性交易做不了；<mark>稀疏索引最细到 8192 行</mark>，单核点查 QPS 只有几百到几千，扛不住详情页；<mark>Mutation 重写整个 part</mark>，频繁状态更新会拖垮集群。**正确架构是双库**：MySQL 做交易库，binlog CDC 同步到 CH 做分析报表，各干各的。"

---

# 第三部分：必背数字表

| 数字 | 含义 |
|---|---|
| <mark>8192</mark> | granule 默认行数（index_granularity） |
| <mark>16384</mark> | 稀疏索引最坏多读的行数（8192×2） |
| <mark>3000</mark> | Too many parts 阈值（parts_to_throw_insert） |
| <mark>1 万行/批</mark> | 批量写入建议，每秒 1~2 次 INSERT |
| <mark>10MB</mark> | index_granularity_bytes 自适应粒度阈值 |
| <mark>150GB</mark> | merge 目标单 part 大小上限 |
| <mark>3 节点</mark> | Keeper 最小集群（多数派容 1 台宕机） |
| <mark>480 秒</mark> | old_parts_lifetime，老 part 延迟删除 |
| <mark>10~50 倍</mark> | FINAL 比 argMax 慢的倍数 |
| <mark>按月</mark> | 分区粒度（别按天） |

---

# 第四部分：一句话金句（面试点睛用）

1. "**排序键加速查询，分区服务数据生命周期。**"
2. "**复制的不是数据，是有序的操作日志。**"
3. "**所有 MergeTree 变体的特殊逻辑都是异步合并时生效的。**"
4. "**右表进哈希表，小表放右边。**"
5. "**写本地表，不写分布式表。**"
6. "**跨分片去重计数，uniqState 存状态，uniqMerge 收口。**"
7. "**复制不能代替备份，误操作会同步到所有副本。**"
8. "**CH 是 OLAP 不是 OLTP，点查和高频小写是它的反模式。**"
9. "**B+Tree 优化随机 IO 次数，稀疏索引优化扫描总量。**"
10. "**引擎选型看数据形态，查询方式必须配合引擎特性。**"

---

## 背诵计划（3 天）

- **第 1 天**：选型 5 张卡 + 总结段 + Q1~Q7
- **第 2 天**：Q8~Q15 + 必背数字表
- **第 3 天**：Q16~Q20 + 10 句金句 + 找人对着镜子模拟问答
