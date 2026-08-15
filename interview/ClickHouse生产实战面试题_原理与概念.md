# ClickHouse 生产实战面试题 —— 原理与概念深度版（扩展 · 含面试官追问）

> 面试官视角：题目按"模块 → 难度（⭐基础 / ⭐⭐进阶 / ⭐⭐⭐专家）"组织。
> 每题结构：**① 标准答案（底层原理）→ ② 🔍 面试官追问与应对（模拟层层递进）→ ③ 踩分点**。
> 资料来源：ClickHouse 官方文档、ClickHouse 学术架构总览（academic-overview）、Altinity 博客/KB、Tinybird、OneUptime、manumartinm 内核实现博客、ClickHouse 源码注释等。

---

# 📌 核心原理图解速记（Part 0）

> 记忆诀窍：用图形锚定概念。下面 12 张图覆盖面试 90% 的原理考点，先看图再背文字。

---

## 图 1：MergeTree Part 磁盘目录结构（Wide 格式）

```
part_202601_1_5_1/              ← 一个 part = 一个目录（不可变）
│
├── primary.idx                 ← ★稀疏主索引：每 granule 首行的排序键值（全量常驻内存）
├── UserID.bin                  ← 列数据：LZ4 压缩块（按排序键排好序）
├── UserID.mrk2                 ← ★mark 文件：granule → .bin 物理偏移的桥梁
├── URL.bin
├── URL.mrk2
├── EventTime.bin
├── EventTime.mrk2
│
├── count.txt                   ← 总行数
├── columns.txt                 ← 列名 + 类型（表结构）
├── checksums.txt               ← 所有文件校验和
├── partition.dat               ← 分区表达式值
├── minmax_EventTime.idx        ← 分区级 min/max（分区剪枝用）
└── skip_idx_x.idx2             ← 跳过索引元数据（如声明了 set/bloom）

记忆口诀：「idx 索引、bin 数据、mrk2 是桥、txt 是元数据」
```

---

## 图 2：稀疏索引 + Granule + Mark 的查询全流程（★★最核心，必背）

```
                          ┌─────────────────────────────────────────┐
   primary.idx (全内存)   │ granule# │ 0    1    2    ... 176 ... 1082│
   每 granule 一条记录    │ UserID   │ 101  205  310  ... 74990001..  │
                          └──────────┴──────────────────────────────-┘
                                              ▲
   查询: SELECT * FROM t WHERE UserID = 749927693
                                              │
                              ① 二分搜索 O(log₂1083)=19 次
                                              │ 找到落在 granule #176
                                              ▼
   UserID.mrk2[176] = { block_offset: 42, granule_offset: 3, rows: 8192 }
                                              │
                              ② mark 文件给出物理位置
                                              ▼
   UserID.bin ──seek(block=42)──► 解压压缩块 ──granule_offset=3──► 读 8192 行
                                              │
                              ③ 流式读取整个 granule（最细粒度）
                                              ▼
                              ④ SIMD 在 8192 行里过滤 UserID=749927693
                                              │
                                              ▼
                                          匹配的行

关键数字：8.87M 行 → 1083 个 granule → primary.idx 仅 96.93KB → 二分 19 次
本质：稀疏索引"为剪枝而生"，不为点查而生 → 最细只到 granule（8192 行）
```

---

## 图 3：向量化执行 vs Volcano 迭代器模型

```
┌─────────────────── Volcano（行式，传统引擎）──────────────────┐
│                                                                │
│   Scan ──next()──► Filter ──next()──► Agg ──next()──► 结果      │
│    ↑        ↑行       ↑         ↑行       ↑          ↑          │
│    │   每行一次虚函数调用，分支预测差，无法 SIMD                │
│   每次只产一行                                                  │
└────────────────────────────────────────────────────────────────┘

┌─────────────────── 向量化（列式，ClickHouse）─────────────────┐
│                                                                │
│   Scan ──[8192行向量]──► Filter ──[向量]──► Agg ──► 结果       │
│           ↑                        ↑                           │
│      紧凑列数组 → 顺序内存访问 → cache 友好                     │
│      → 编译器/手写 AVX-512 → SIMD 并行                          │
│      每批一次虚调用（摊薄 1000×）                                │
└────────────────────────────────────────────────────────────────┘

记忆：「列存是向量化的前提，向量化是 SIMD 的前提，三者天生一对」
```

---

## 图 4：三层并行（SIMD → 多核 → 多节点）

```
                              查询到来
                                 │
         ┌───────────────────────┼───────────────────────┐ ③ 多节点层
         ▼                       ▼                       ▼  （分片级）
      分片1                    分片2                    分片3
   partial聚合                partial聚合              partial聚合
         │                       │                       │
   ┌─────┼─────┐           ┌─────┼─────┐           ┌─────┼─────┐ ② 多核层
   ▼     ▼     ▼           ▼     ▼     ▼           ▼     ▼     ▼ （pipeline lane）
  核1   核2   核N          核1   核2   核N          核1   核2   核N
   │     │     │            │     │     │            │     │     │
   ▼     ▼     ▼            ▼     ▼     ▼            ▼     ▼     ▼ ① SIMD 层
  AVX-512(对一批数据做 SIMD 运算)                                    （数据元素级）
   └─────┴─────┘           └─────┴─────┘           └─────┴─────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 ▼
                    initiator: GroupStateMerge 合并部分状态
                                 │  （只传聚合状态，不传亿行明细！）
                                 ▼
                              最终结果

记忆顺序：「单条数据 SIMD → 单机多核 → 多机多分片」自底向上三层叠加
```

---

## 图 5：Merge 过程（LSM 风格的渐进合并）

```
   INSERT   INSERT   INSERT   INSERT
     │        │        │        │
     ▼        ▼        ▼        ▼
  [part1] [part2] [part3] [part4]        ← 每次插入 = 1 个 part（不可变、已排序）
     └──┬───┘      └───┬───┘
        ▼                ▼
    [part_A 较大]   [part_B 较大]        ← 后台 merge: k-way 归并排序 + 重建索引
        └────────┬───────┘
                 ▼
         [part_超大 ~150GB]              ← 渐进逼近全局有序（但永不保证跨 part 全局有序）

  ⚠ 规则：① 不同 partition 永不合并
          ② 老 part 经 old_parts_lifetime(480s) 才物理删
          ③ 合并是确定性的 → 副本可共享结果 part（省 CPU）

  ⚠ 红线：单分区 active part > ~1000 → "Too many parts" 报错 → 必须攒批写入
```

---

## 图 6：复制日志机制 + Keeper znode 结构

```
                ┌─────── ClickHouse Keeper 集群（Raft，3 节点）──────┐
                │                                                     │
   路径:        │  /clickhouse/tables/{shard}/{table}/                │
                │  ├── log/                  ← ★复制日志（追加写）     │
                │  │   ├── entry#1: INSERT part X                     │
                │  │   ├── entry#2: MERGE X+Y → Z (assign r1)         │
                │  │   └── entry#3: MUTATION ...                      │
                │  ├── replicas/                                        │
                │  │   ├── r1/                                          │
                │  │   │   ├── log_pointer = 3   ← 已消费到第3条       │
                │  │   │   ├── queue/          ← 本地待执行任务        │
                │  │   │   ├── columns/        ← 表结构               │
                │  │   │   └── minmax/                                │
                │  │   └── r2/                                          │
                │  │       ├── log_pointer = 2   ← ⚠落后1条(副本延迟) │
                │  │       └── queue/                                 │
                │  └── temp/                                           │
                └─────────────────────────────────────────────────────┘

   写入时序（异步，最终一致）:
   Client ──INSERT──► r1 ──① 本地落盘 part
                          ──② 追加 log entry 到 Keeper
                          ──③ 返回 Client 成功（不等 r2）
                                          ↓ Keeper watch 通知
                                       r2 ──④ 拉取 entry
                                          ──⑤ 从 r1 fetch part（不重算）
                                          ──⑥ 本地落盘，log_pointer++

   关键认知：副本是【最终一致】，非强一致；合并结果可跨副本共享（确定性）
```

---

## 图 7：分片 + 复制拓扑（两个正交维度）

```
                    ┌──────────────────────────┐
                    │  Distributed 表（逻辑视图）│  ← 客户端查询入口
                    └────────────┬─────────────┘
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
          【分片1】           【分片2】           【分片3】     ← ① 分片：水平扩展
       (hash=key%3=0)      (hash=key%3=1)      (hash=key%3=2)    数据量+吞吐
          ┌──┴──┐            ┌──┴──┐            ┌──┴──┐
          ▼     ▼            ▼     ▼            ▼     ▼
         r1    r2            r1    r2            r1    r2       ← ② 复制：高可用
       (主)  (备)          (主)  (备)          (主)  (备)        +读扩展
         │  ╱  │            │  ╱  │            │  ╱  │           (ReplicatedMergeTree
         │ ╱   │  Keeper    │ ╱   │  Keeper    │ ╱   │  Keeper    多主都可读写)
         ╱     ╲            ╱     ╲            ╱     ╲
       Keeper集群(3节点)  共享协调           跨分片独立

   分片与复制【正交】：可单独使用
   • 只分片不复制：数据量大但容忍单点故障
   • 只复制不分片：高可用但单分片容量上限
   • 两者都做：生产标配
```

---

## 图 8：JOIN 算法对比 —— hash vs grace_hash vs direct

```
┌────────── hash JOIN（默认，右表进内存）──────────┐
│   右表(小) ────────► [内存哈希表]                   │
│                            ▲                       │
│   左表(大) ────probe───────┘ ───► 结果             │
│   ⚠ 右表太大 → OOM                                 │
└────────────────────────────────────────────────────┘

┌────────── grace_hash JOIN（分桶落盘，省内存）──────┐
│   右表 ─hash─► [桶0][桶1]...[桶N] ─落盘─┐           │
│   左表 ─hash─► [桶0][桶1]...[桶N] ─落盘─┤           │
│                                          ▼          │
│   逐桶: 桶i右表→内存哈希 ←probe← 桶i左表 → 结果    │
│   桶太大 → 再哈希分裂（桶数翻倍，max 1024）         │
│   ✅ 大表 JOIN 大表不 OOM                           │
└────────────────────────────────────────────────────┘

┌────────── direct JOIN（用右表排序键点查，0内存）───┐
│   左表每行 ──► 右表(ReplicatedMergeTree)            │
│               按排序键二分稀疏索引 ──► 读 granule    │
│   ✅ 右表超大的维度表也省内存                        │
│   ⚠ 左表大时慢（每行一次右表查找）                  │
└────────────────────────────────────────────────────┘

记忆：「小右表→hash；大对大→grace；右表是排序键大表→direct」
```

---

## 图 9：冷热分层存储 + TTL 全景

```
   时间轴（数据年龄）─────────────────────────────────────────►

   ┌─────────────┬─────────────────┬─────────────────────────┐
   │  近 7 天     │   7~90 天        │   90 天 ~ 365 天          │
   │  🔥 热数据    │   🌡 温数据       │   ❄ 冷数据                │
   ├─────────────┼─────────────────┼─────────────────────────┤
   │  本地 NVMe   │   本地 SSD        │   S3 对象存储 + s3_cache  │
   │  CODEC(LZ4)  │   RECOMPRESS      │   RECOMPRESS             │
   │              │   CODEC(ZSTD(1))  │   CODEC(ZSTD(5))         │
   └─────────────┴─────────────────┴─────────────────────────┘
        ▲               ▲                          ▲
        │               │                          │
   TTL dt+7d        TTL dt+90d                 TTL dt+365d
   RECOMPRESS       TO VOLUME 'cold'           DELETE  ← 行级删除

   storage_policy 配置: hot_volume(NVMe) → cold_volume(S3)，MOVE 自动迁移
   ⚠ TTL 是异步 merge 时执行，非实时；DROP PARTITION 比 TTL DELETE 快几个数量级
```

---

## 图 10：AggregatingMergeTree + 物化视图 预聚合数据流

```
   ┌──────────────────────────────────────┐
   │  明细表 source（MergeTree）           │  ← 原始事件，亿级行/天
   │  event_id, uid, dim, amount, ts       │
   └──────────────────┬───────────────────┘
                      │ 每次 INSERT 自动触发（增量，不重算历史）
                      ▼
   ┌──────────────────────────────────────────────────────┐
   │  CREATE MATERIALIZED VIEW mv TO agg_target AS         │
   │  SELECT dim,                                          │
   │         uniqState(uid)   AS uv,    ← AggregateFunction │
   │         sumState(amount) AS amt    ← 可合并的聚合状态   │
   │  FROM source GROUP BY dim                             │
   └──────────────────┬───────────────────────────────────┘
                      │ 写入聚合状态
                      ▼
   ┌──────────────────────────────────────────────────────┐
   │  agg_target（AggregatingMergeTree）                   │  ← 万级行（按 dim 压缩）
   │  合并时自动 merge 聚合状态（uniqMerge / sumMerge）     │
   └──────────────────┬───────────────────────────────────┘
                      │ 查询
                      ▼
   SELECT dim, uniqMerge(uv), sumMerge(amt) FROM agg_target GROUP BY dim
                      │
                      ▼
              毫秒级响应（数据已预聚合）

   记忆：「明细表 → MV 增量聚合 → AggregatingMergeTree 存状态 → 查询 Merge」
```

---

## 图 11：Mutation 执行流程（重量级，慎用）

```
   ALTER TABLE t DELETE WHERE ts < '2026-01-01'
                      │
                      ▼
   ┌──────────────────────────────────────────┐
   │ system.mutations 队列（mutation_id 自增） │  ← 可监控 is_done
   │   mutation#5: DELETE WHERE ts < '...'     │
   └──────────────────┬───────────────────────┘
                      │ 后台线程，复用 merge 机制
                      ▼
        ┌─────────────┴──────────────┐
        ▼              ▼              ▼
   [part_1]       [part_2]       [part_9*]     ← *mutation 后新插入的，跳过
   读全部行        读全部行
   应用谓词        应用谓词                     ← DELETE: 删匹配行; UPDATE: 改列
   写新 part_1'   写新 part_2'
        │              │
        └──────┬───────┘
               ▼
        新 part 替换老 part（老 part 经 480s 删除）
                      │
                      ▼
        ⚠ 重写整个 part → I/O+CPU 极重
        ⚠ 与 merge 争抢 background_pool
        ⚠ 大表可能跑几小时~几天，卡后续 mutation
        ⚠ 不可回滚（KILL 不撤已写 part）
        ✅ 替代：Replacing/CollapsingMergeTree、删分区、轻量级 DELETE
```

---

## 图 12：分布式聚合下推（跨分片聚合快的核心）

```
   Client ──SELECT region, SUM(amount) FROM dist_table GROUP BY region──► initiator
                                                                            │
                  ① 查询下推到各分片（partial aggregation）                  │
        ┌───────────────────────────────────────────────────────────────────┘
        ▼                          ▼                          ▼
     分片1                       分片2                       分片3
   扫本分片数据                扫本分片数据                扫本分片数据
   GROUP BY region             GROUP BY region             GROUP BY region
   SUM(部分)                   SUM(部分)                   SUM(部分)
   ↓ 只传【部分聚合状态】       ↓                          ↓
   ↓（少量行，非亿行明细）      ↓                          ↓
        │                          │                          │
        └──────────┬───────────────┴──────────────────────────┘
                   ▼  ② initiator 合并
            GroupStateMerge(部分状态)
                   │
                   ▼  ③ 最终 SUM 结果
                Client

   关键：聚合下推 → 网络只传 N 个分组的状态（如 30 个 region），不传亿行明细
   ⚠ count(distinct) 不可直接下推 → 必须用 uniqState/uniqMerge（HLL 状态可合并）
```

---

# 📦 第一部分：存储引擎与 MergeTree 内核

## Q1（⭐）请完整描述一条 INSERT 数据落盘后，在 MergeTree 引擎内部发生了什么？

**① 标准答案：**

1. **构建 Part**：每次 INSERT 形成一个独立 data part（一个目录）。若跨 partition 会拆成多个 part。part 内部行按 `ORDER BY` 做字典序全排序（在内存中完成）。
2. **格式选择（Wide / Compact）**：当 part 行数 < `min_rows_for_wide_part`（默认 0）或字节 < `min_bytes_for_wide_part`（默认 10MB）→ **Compact**（所有列挤在一个 `data.bin` + 一个 `data.mrk3`）；否则 **Wide**（每列一个 `列名.bin` + `列名.mrk2`）。Compact 省小文件元数据开销。
3. **列式压缩写盘**：每列按"压缩块"写 `.bin`，块大小受 `min_compress_block_size`（默认 65536）和 `max_compress_block_size`（默认 1048576）控制；默认 LZ4。每个 granule 的列数据流式追加到压缩块。
4. **生成 Granule 与稀疏主索引**：列按 **granule = 8192 行**（默认 `index_granularity`）逻辑切分；自适应粒度下，当 granule 字节 ≥ `index_granularity_bytes`（默认 10MB）也提前切分。每个 granule 首行的排序键值写入 `primary.idx`（未压缩平坦数组，**全量常驻内存**）；每列 `.mrk2` 记录该 granule 在 `.bin` 中的 `(block_offset, granule_offset, rows_count)` 三元组。
5. **辅助文件**：`count.txt`（总行数）、`columns.txt`（列结构）、`checksums.txt`（校验）、`partition.dat` + `minmax_{col}.idx`（分区剪枝元数据）。
6. **后台 Merge**：独立线程池把小 part 按 k-way 归并成大 part（目标上限 ~150GB，`max_bytes_to_merge_at_max_space_in_pool`）。**不同 partition 永不合并**。
7. **旧 part 清理**：合并/变异后老 part 经 `old_parts_lifetime`（默认 480s）才物理删。

**② 🔍 面试官追问：**

- **追问 1：为什么不在插入时就排序好整个表，而要 part 级排序？**
  答：ClickHouse 是 LSM 思想（log-structured merge）。全局排序需要锁表或巨大临时空间；part 级排序让写入变成纯追加，写吞吐极高；最终的全局有序性靠后台 merge 渐进逼近（但**永远不保证跨 part 全局有序**——这是重要认知）。

- **追问 2：`primary.idx` 为什么不放磁盘？放内存不会浪费吗？**
  答：放内存是为了**二分搜索零 IO**。它的设计目标就是"小到全内存"：8.87M 行只要 96.93KB。如果它比可用内存大，ClickHouse 直接**启动报错**而非退化到磁盘——因为退化后查询性能会断崖式下降，违背设计哲学。

- **追问 3：自适应粒度（adaptive granularity）解决了什么问题？**
  答：固定 8192 行粒度下，若单行很大（如 String 列很长），一个压缩块会非常大，解压时一次性读很多字节浪费 IO。自适应粒度按"行数 OR 字节数"双阈值切分，让变长列场景下的 granule 大小可控，解压单元更合理。

- **追问 4：part 目录下有哪些文件？Compact 和 Wide 各是什么？**
  答：Wide 模式每列 `col.bin` + `col.mrk2`；Compact 模式共享 `data.bin` + `data.mrk3`。两者都有 `primary.idx`、`count.txt`、`columns.txt`、`checksums.txt`、`partition.dat`、`minmax_*.idx`。Projection 在子目录里另存一份排好序的数据。

**③ 踩分点：** part 不可变；排序在 part 内非全局；主索引每 granule 一条；新 part 不被旧 mutation 触碰；merge 是渐进的。

---

## Q2（⭐⭐）ClickHouse 的主索引为什么是"稀疏的"？与 InnoDB B+Tree 的本质区别？

**① 标准答案：**

| 维度 | InnoDB B+Tree | ClickHouse 稀疏主索引 |
|---|---|---|
| 索引粒度 | 每行一条 | 每 granule（8192 行）一条 |
| 索引大小 | 与行数线性，TB 表→GB 级 | 8.87M 行 → 1083 条 → 96.93KB |
| 常驻位置 | 索引页常驻缓冲池（部分磁盘） | **完全常驻内存**，超内存直接报错 |
| 定位方式 | B+Tree 二分 + 多次磁盘 seek（log₂n） | 二分稀疏索引圈 granule → 流式读 8192 行 |
| 写入代价 | 插入需平衡、页分裂 | 追加写新 part |
| 点查能力 | 极强（QPS 数万~数十万/核） | 弱（每查询扫 8192 行，QPS 上限低） |
| 范围扫描/聚合 | 弱（行存，列读取放大） | **极强**（列存+压缩+向量化） |

**本质**：B+Tree 用"每行精确索引 + 多次 seek"换点查；ClickHouse 放弃单行精度，换"索引全内存 + 顺序 IO + 列存 + 压缩 + 向量化"组合拳，把**扫描和聚合**做到极致。它不是为点查设计的索引，而是**为剪枝（pruning）设计的索引**。

**② 🔍 面试官追问：**

- **追问 1：那 ClickHouse 到底能不能做点查？性能怎样？**
  答：能。靠二分稀疏索引圈定 1~2 个 granule，再 SIMD 在 8192 行里过滤。单点查延迟通常毫秒级，但**单核 QPS 上限低**（因为每个查询都要流式扫 8192 行），不适合高并发点查（如电商详情页）。

- **追问 2：稀疏索引最坏情况多读多少行噪音？**
  答：单个数据块可能多读 `index_granularity * 2` 行（即最坏 16384 行）。官方文档明确写了这个上界。

- **追问 3：能不能让 ClickHouse 点查更快？**
  答：①建 **Projection** 改变排序键让点查列在前；②用 `low_cardinality` + 跳过索引；③用 **ReplicatedMergeTree + direct JOIN**；④极端情况下用外部缓存（Redis）。但根本上 CH 不适合当 KV 用。

- **追问 4：为什么稀疏索引能放内存，B+Tree 不能？**
  答：B+Tree 索引和数据耦合（聚簇索引），每行都要有索引项指向数据，行数和索引项数 1:1。稀疏索引是"每块一条"，N 行只需要 N/8192 条，压缩了 8000+ 倍。

**③ 踩分点：** 索引全内存；为剪枝而非点查设计；与行存本质差异在 IO 模式（顺序 vs 随机 seek）。

---

## Q3（⭐⭐⭐）主索引的"二分搜索"与"通用排除搜索"区别？排序键顺序设计铁律？

**① 标准答案：** 查询分两阶段：**Stage 1 圈选 granule → Stage 2 读数据**。

- **二分搜索（Binary Search）**：过滤命中**排序键第一列**时，该列在 granule 边界单调递增，对 `primary.idx` 做 O(log₂n) 二分。8.87M 行（1083 mark）只需 **19 次比较**。
- **通用排除搜索（Generic Exclusion Search）**：过滤命中**排序键第二列及以后**，ClickHouse 逐个 granule 检查"该 granule 的列范围是否可能含目标值"。**有效性严重依赖前置列基数差异**：
  - 前置列基数**低** → 同值跨多 granule → 后续列在这些 granule 内单调 → 高效排除。
  - 前置列基数**高** → 每 granule 的前置列值都变 → 后续列标记不单调 → 退化为近全表扫。官方实测 `(UserID, URL)` 表查 URL：1083 个 granule 选了 **1076 个**。

**排序键设计铁律：**
1. 第一列必须是**过滤性最强、查询最常用**的列。
2. 低基数列在前，高基数在后。
3. 时间列通常靠后（靠分区剪枝）。
4. 绝不把高基数随机列（如 `event_id`）放首位。

**② 🔍 面试官追问：**

- **追问 1：我有个日志表，常按 `app_name + user_id + event_time` 查，排序键怎么设？**
  答：`(app_name, user_id, event_time)`。app_name 低基数在前（前置列单调性好），user_id 高基数在中（在前置列范围内可二分/排除），event_time 末位（时间范围靠分区 + 末位排除）。

- **追问 2：如果业务两个维度查询频率相当，比如既按 user_id 查又按 device_id 查，怎么办？**
  答：排序键只能选一个最优，另一个用 **Projection**（建一份按另一排序键排好序的数据副本）。或用跳过索引（set/bloom）缓解。Projection 代价是存储翻倍 + 写入变慢。

- **追问 3：排序键列越多越好吗？**
  答：不是。列越多，主索引条目越大（每条记录所有列值），内存占用上升；且写排序成本上升。通常 3~5 列足够，靠 Projection 补充。

- **追问 4：`ORDER BY tuple()` 是什么意思？什么时候用？**
  答：无排序键，数据按插入序存。只适合纯追加、无点查/范围查的场景（如纯日志归档），查询只能全表扫。

**③ 踩分点：** 第一列决定查询效率上限；基数差异是 generic exclusion 失效的根因；Projection 是排序键冲突的解药。

---

## Q4（⭐⭐）数据跳过索引（Data Skipping Index）有哪几种？bloom_filter 为什么不能优化 `!=`？

**① 标准答案：** 跳过索引在 granule 之上加轻量元数据，`INDEX idx_name expr TYPE xxx GRANULARITY n` 声明，`n` 表示跨 n 个 granule 聚合（默认 1）。

| 类型 | 原理 | 适用 | 注意 |
|---|---|---|---|
| `minmax` | 记录 min/max | 数值/时间，列大致有序 | 对乱序列几乎无效 |
| `set(max_rows)` | 记录最多 max_rows 唯一值 | 低局部基数列 | max_rows=0 存全部 |
| `bloom_filter([fpr])` | 布隆过滤器，默认误判率 0.025 | 等值/IN 高基数列 | **only-negative**，不能证存在 |
| `tokenbf_v1` / `ngrambf_v1` | token/n-gram 布隆 | 字符串 LIKE | 已逐步被倒排索引取代 |
| `text` | 倒排索引（v26.2 GA） | 全文检索 | 推荐替代 tokenbf |
| `vector_similarity` | ANN | 向量相似度 | 较新特性 |

**bloom 不能优化 `!=`/`NOT IN`/`NOT LIKE` 的本质**：布隆只能给"**可能存在**"或"**一定不存在**"。`!= x` 这种**期望结果为真**的谓词，布隆说"可能存在 x"无法证伪，所以不能据此排除 granule。

**② 🔍 面试官追问：**

- **追问 1：跳过索引和主索引是什么关系？会冲突吗？**
  答：主索引是**强制的、基于排序键的稀疏索引**；跳过索引是**可选的补充**，用于非排序键列。两者协作：先用主索引剪枝，再用跳过索引在剩余 granule 上二次剪枝。不冲突。

- **追问 2：`GRANULARITY n` 设大好还是小好？**
  答：`n=1` 表示每个 granule 一组元数据，剪枝最精细但元数据开销大；n 越大越省空间但剪枝越粗。低基数列适合大 n，高基数列适合 n=1。

- **追问 3：跳过索引会拖慢写入吗？**
  答：会。每个跳过索引在插入和合并时都要维护元数据。索引越多写入越慢、part 越大。生产里通常只建 2~3 个高价值跳过索引。

- **追问 4：minmax 索引和列本身的 minmax 元数据（`minmax_{col}.idx`）有什么区别？**
  答：后者是**分区级**的自动元数据（用于分区剪枝），不需要声明；前者是**用户声明**的 granule 组级（多 granule）元数据，用于 part 内剪枝。

**③ 踩分点：** 跳过索引只剪枝不过滤；布隆 only-negative；索引有写入代价，不是越多越好。

---

# ⚡ 第二部分：执行引擎

## Q5（⭐⭐）什么是向量化执行？与 Volcano 模型、codegen 的优劣？

**① 标准答案：**

- **Volcano 迭代器模型**（传统行式）：`open/next/close`，每次 `next` 返回一行，每算子一次虚函数调用。优点：实现简单、流式低内存；缺点：**每行一次虚调用、分支预测差、无法 SIMD**。
- **向量化执行**（ClickHouse、MonetDB/X100）：算子间传递**整批列（chunk，默认按块）**而非单行。优点：①虚调用摊薄到每批一次（降 1000×）；②紧凑列 → 顺序内存访问、cache 友好；③紧凑列 → SIMD（AVX2/AVX-512）。
- **代码生成（codegen，如 Hyper/Impala）**：把整条算子链编译成一个函数，消除虚调用、中间数据放寄存器。ClickHouse 是**混合策略**：默认向量化，对**重复出现的表达式**做 LLVM 编译（机会主义 codegen），相邻算子融合成一个编译单元。

**ClickHouse 运行时挑内核**：同一段向量化代码编译多版本（非向量化 / AVX2 自动向量化 / 手写 AVX-512），按 CPU 能力选最快。最低要求 SSE 4.2。

**② 🔍 面试官追问：**

- **追问 1：为什么不全用 codegen？**
  答：LLVM 编译一个查询要几十~几百 ms，对短查询是纯开销；向量化对短查询更友好、代码可维护性高。ClickHouse 选择"向量化为主 + 高频表达式 codegen"的混合，兼顾两端。

- **追问 2：向量化执行是不是就不会有虚函数调用了？**
  答：还有，但摊薄到每批一次。ClickHouse 进一步用 codegen 把热点算子链融合，进一步消除虚调用。完全消除虚调用需要把整条 pipeline 编译。

- **追问 3：向量化执行和列存是什么关系？必须列存才能向量化吗？**
  答：列存是向量化的**前提**——同质数据紧凑存放才能批量 SIMD 处理。行存里同一列的值分散在不同行，无法高效向量化。两者是天生一对。

- **追问 4：一个 batch（chunk）多大合适？**
  答：ClickHouse 默认按压缩块/granule 处理，典型几千行。太小虚调用摊薄不足；太大 cache miss、寄存器压力。这是一个经验权衡值。

**③ 踩分点：** 向量化 = 批处理 + 列存 + SIMD；codegen 是补充不是替代；混合策略是 CH 的工程智慧。

---

## Q6（⭐⭐⭐）ClickHouse 有哪几层并行？分布式查询如何切片执行？

**① 标准答案（官方 academic-overview，三层并行）：**

1. **SIMD 层（数据元素级）**：向量化内核对一批数据做 SIMD。
2. **多核层（数据块级）**：物理算子按核数展开成多个**执行 lane**，worker 线程并行推进。横向（同阶段不同输入）+ 纵向（未被 pipeline breaker 阻断的相邻阶段）并行。**Exchange 算子（如 Repartition）动态路由 chunk 做负载均衡**。并行度可**查询中途**（1 ~ max_threads）动态调整。
3. **多节点层（分片级）**：initiator 把工作下推到远端节点。远端处理程度：①只流回原始列；②过滤后回传；③**过滤+聚合后回传部分聚合状态**（关键优化）；④整个查询远端跑完。initiator 用 `GroupStateMerge` 合并各分片部分状态。

**② 🔍 面试官追问：**

- **追问 1：聚合为什么能下推？所有聚合都能下推吗？**
  答：可加聚合（sum/count/avg/min/max）能下推——部分聚合状态可合并。**不可加聚合（如 count distinct）不能直接下推**，要用 `uniqState`/`uniqHLL12` 把状态变成可合并的形式（HLL/HyperLogLog 的状态可合并）。

- **追问 2：`max_threads` 设多少合适？**
  答：默认是 CPU 核数。但 CH 是**并行扫描型**，单查询会吃满所有核；高并发场景要调小（如 `max_threads=8`），否则几个大查询就把机器打满，互相争抢。生产常按负载类型分组设 profile。

- **追问 3：什么是 pipeline breaker？为什么影响并行？**
  答：pipeline breaker 是必须**收齐全部上游数据**才能产出的算子（如排序、完整聚合、某些 JOIN）。它阻断纵向并行（上游和下游不能同时跑），强制落盘或缓存中间结果。CH 用 Exchange 算子做 repartition 来缓解。

- **追问 4：跨分片查询的网络开销怎么优化？**
  答：①聚合下推减少传输；②用 `prefer_localhost_replica` 优先本地副本；③维度表复制到每分片避免分布式 JOIN；④`distributed_aggregation_memory_efficient` 减少中间状态；⑤网络带宽监控 + 分片数合理（不是越多越好）。

**③ 踩分点：** 三层并行；聚合下推是分布式聚合快的核心；pipeline breaker 限制纵向并行；count distinct 要用 State 函数。

---

# 🌐 第三部分：分布式、复制与 Keeper

## Q7（⭐⭐）分片和复制的关系？Keeper/ZooKeeper 的角色？

**① 标准答案：**

- **分片（正交）**：通过分片表达式把逻辑表切到多节点。`Distributed` 表引擎提供全局视图。用途：①数据超单节点容量（数十 TB）；②分散负载。
- **复制（正交）**：每个 MergeTree 子引擎有 `ReplicatedMergeTree` 变体，基于 **Raft 共识的多主协调**。每个 MergeTree 子引擎（Replacing/Summing/Collapsing/Aggregating）都有 Replicated 版本。
- **Keeper/ZooKeeper 角色**：维护**全局复制日志**。表状态 = "一组 part + 元数据"。插入、合并、mutation、DDL 记录为状态转换日志条目，写入 Keeper 集群（推荐 3 节点）。副本**异步回放** → **最终一致**。

**② 🔍 面试官追问：**

- **追问 1：为什么复制是最终一致而不是强一致？**
  答：异步回放日志换取高写入吞吐和可用性。强一致需要同步等所有副本（quorum insert），会显著降吞吐、放大尾延迟，副本故障时写入阻塞。OLAP 场景通常能容忍秒级延迟。

- **追问 2：ClickHouse Keeper 和 ZooKeeper 能混用吗？**
  答：协议兼容（Keeper 实现了 ZK 协议），可替换，但**同一集群不要混用**——两者在事务语义、watch 实现上有细微差异，混用易出诡异 bug。新部署推荐 Keeper（C++、无 JVM、省内存、同二进制打包）。

- **追问 3：`insert_quorum` 是什么？什么时候用？**
  答：同步写入到 N 个副本才返回成功，换取读强一致（配合 `select_sequential_consistency`）。代价是写入延迟升高、副本故障时写阻塞。仅在金融级一致性场景用，平时不开。

- **追问 4：新加副本怎么同步数据？从头回放日志吗？**
  答：不是。新副本**直接从已有副本拷贝 part**（fetch part），只回放日志尾部增量。否则历史日志可能已被清理（Keeper 日志有保留上限），且重放太慢。

**③ 踩分点：** 分片复制正交；最终一致是设计选择；Keeper 是 ZK 的 C++ 替代；新副本靠 fetch part 而非重放。

---

## Q8（⭐⭐⭐）分布式表（Distributed）查询时，GLOBAL JOIN/GLOBAL IN 的陷阱？写本地表 vs 分布式表？

**① 标准答案：**

- **写本地表 vs 分布式表**：生产推荐**直接写本地表**（每分片的 Replicated 表）。写 `Distributed` 表由接收节点按分片键转发，多一跳网络、写吞吐受单节点限制、节点宕机会丢未转发数据（Distributed 写非事务）。
- **GLOBAL IN / GLOBAL JOIN 陷阱**：
  - 普通 `IN` 子查询在分布式表上**每分片独立执行**，若子查询里有非分片键的本地表，每分片只看本分片数据 → **结果不正确**或重复算。
  - `GLOBAL IN` 让 initiator **先算一次**子查询，结果集广播到所有分片再执行，保证正确性。
  - **但 GLOBAL 仍有陷阱**：广播结果集可能很大（撑爆内存/网络）；且 GLOBAL 子查询在 initiator 单点执行，无法下推并行。最佳实践仍是**把维度表复制到每分片**。

**② 🔍 面试官追问：**

- **追问 1：维度表多大才适合复制到每分片？**
  答：经验上 < 1GB、更新不频繁的维度表适合复制（用 Replicated 表 + `Distributed` 或直接每分片都建）。太大则考虑 partial_merge/grace_hash JOIN。

- **追问 2：写 Distributed 表配 `insert_distributed_sync=1` 是不是就安全了？**
  答：sync 模式下写入会等所有分片确认，安全性提升但仍非事务（部分分片成功部分失败会留下不一致）。且 sync 模式吞吐大幅下降。生产仍优先写本地表 + 客户端分片。

- **追问 3：count distinct 跨分片怎么算？**
  答：用 `uniqState(uid)` 存储中间状态（HLL/HyperLogLog），跨分片用 `uniqMerge(state)` 合并。直接 `count(distinct uid)` 跨分片会先把所有 uid 拉到 initiator 再算，内存爆炸。

- **追问 4：Distributed 表查询会不会重复计算？**
  答：聚合是两阶段（partial 下推 + final 合并），不重复。但**非聚合的纯 SELECT**会把所有匹配行拉到 initiator，可能重复（取决于查询）。要靠 `enable_optimize_predicate_expression` 等优化下推过滤。

**③ 踩分点：** 写本地表；GLOBAL 解决正确性但有内存/单点陷阱；维度表复制是 JOIN 最优解；count distinct 必须用 State 函数。

---

## Q9（新增 ⭐⭐⭐）请详细讲一下 ReplicatedMergeTree 的复制日志机制、log pointer、以及副本延迟（replica lag）的根因与缓解。

**① 标准答案：**

**复制日志机制：**
1. 每个表在 Keeper 有 znode 路径 `/clickhouse/tables/{shard}/{table}/replicas/{replica}`。
2. 插入/合并/mutation 在 Keeper 的 **log** znode 追加条目（Log Entry）。
3. 各副本通过 Keeper **watch** 感知新条目，**异步**拉取回放：
   - 插入：从源副本拉 part 或本地回放。
   - 合并：本地重算，或从别的副本拉结果 part（`fetch_merged_part`，避免重复算）。
4. 副本维护 **log pointer**（已消费到第几条）。
5. Keeper 还维护：`columns`（结构）、`minmax`、`queue`（待执行任务队列）等 znode。

**副本延迟根因：**
1. **网络瓶颈**：拉 part 带宽打满。
2. **副本机器慢**：CPU/IO 跟不上回放。
3. **大 part/大量 mutation**：单回放任务太重。
4. **Keeper 抖动**：watch 通知滞后。
5. **合并堆积**：副本本地合并跟不上，part 多，回放更慢。

**监控**：`system.replicas` 的 `absolute_delay`（秒）、`log_pointer`、`queue_size`。

**缓解**：扩 Keeper/换 Keeper；加副本机器 IO/CPU；减少大 mutation（用 TTL/分区删代替）；控制 part 大小和写入批；读时避开高延迟副本（`load_balancing`）。

**② 🔍 面试官追问：**

- **追问 1：log 在 Keeper 里会无限增长吗？**
  答：不会。CH 有 log 清理机制（`log_rotation_size`、`log_rotation_period`），定期截断老条目。但前提是所有副本都已消费——所以**长期离线的副本恢复时可能 fetch 不全日志**，要靠 fetch part 补数据。

- **追问 2：合并为什么可以让副本拉结果 part 而不是各自重算？**
  答：合并是**确定性的**（相同 part 集合 + 相同合并算法 → 相同结果），CH 给每个合并分配唯一 ID，一个副本算完把结果 part 上传，其他副本直接拉。这叫 **assignment of merges**，大幅省 CPU。

- **追问 3：两个副本同时触发同一个合并会冲突吗？**
  答：不会。合并任务通过 Keeper **协调分配**，只有一个副本执行，其他副本拉结果。Keeper 的 watch + 临时 znode 保证分配唯一。

- **追问 4：副本延迟几十秒，业务读到了旧数据怎么办？**
  答：①监控告警 `absolute_delay`；②读关键场景用 `insert_quorum` + `select_sequential_consistency`；③读副本选择策略避开延迟副本；④根本上是降低延迟（加机器、减 mutation）。

**③ 踩分点：** 复制是异步最终一致；合并可跨副本共享结果；log 有保留上限；延迟监控 `system.replicas`。

---

# 🔗 第四部分：JOIN 引擎

## Q10（⭐⭐⭐）ClickHouse 有哪些 JOIN 算法？各场景与内存特性？

**① 标准答案（默认 `join_algorithm = hash`）：**

| 算法 | 原理 | 内存 | 适用 |
|---|---|---|---|
| **hash**（默认） | 右表建哈希表内存，左表流式探测 | 右表全量进内存 | 右表小；支持 ASOF/不等式 |
| **parallel_hash** | 右表切片，多线程并行建多哈希表 | 同 hash，建表更快 | 右表较大但能放内存 |
| **direct** | 不建哈希，用右表**主键/排序键**点查 | 几乎 0 内存 | 右表是 ReplicatedMergeTree 且 JOIN 键=排序键，左表小 |
| **partial_merge** | 右表按 JOIN 键排序落盘，左表分批归并 | 右表排序磁盘+小块内存 | 右表大；仅支持 ALL 严格度 |
| **full_sorting_merge** | 双方排序后归并 | 排序缓冲 | 左右都大、键大致有序；支持 ASOF |
| **grace_hash** | 哈希分桶落盘，分批 probe | 可控（按桶数），`grace_hash_join_max_buckets` 默认 1024 | 大 JOIN 大表，内存压力 |
| **auto** | 先 hash，内存超阈值降级 merge | 动态 | 省心 |
| **full** | hash + grace 组合 | 中等 | 通用 |

**严格度（join_default_strictness）**：`ALL`（笛卡尔积）/ `ANY`（去积）/ `SEMI`（白名单）/ `ANTI`（黑名单）/ `ASOF`（最近时间匹配）。

**② 🔍 面试官追问：**

- **追问 1：grace_hash 的"分桶"具体怎么工作？**
  答：①把左右表按 JOIN 键哈希到 N 个桶（bucket），各自落盘；②逐桶处理：第 i 桶的右表建内存哈希表，左表第 i 桶探测；③某桶还是太大就再哈希分裂（桶数翻倍，最多到 `grace_hash_join_max_buckets`）。本质是用磁盘换内存，类似外部排序的归并思想。

- **追问 2：direct JOIN 为什么省内存？原理是什么？**
  答：右表是 MergeTree，JOIN 键就是排序键时，左表每行直接对右表做**稀疏索引二分定位 + granule 读取**，不需要建哈希表。内存几乎为 0。代价是左表每行都要一次右表查找，左表大时慢。适合"大右表 + 小左表 + 键=排序键"。

- **追问 3：`partial_merge` 为什么只支持 ALL 严格度？**
  答：partial_merge 把右表排序后归并，归并过程中难以维护"是否已匹配"的状态来支持 SEMI/ANTI/ANY 的去重语义。这些严格度需要在内存里记录匹配状态，违背了 partial_merge 的"省内存"初衷。

- **追问 4：JOIN 时左右表顺序重要吗？**
  答：重要。CH 把**右表建哈希表**，所以小表放右边。新版本有 runtime join reordering 自动交换，但 RIGHT/FULL JOIN 不能随意交换，此时务必保证右表能进内存或用 grace_hash。

- **追问 5：生产里 JOIN OOM 了怎么救？**
  答：①切 `join_algorithm='grace_hash'`；②降 `max_memory_usage` 让它早点溢写；③把维度表复制到每分片做本地 JOIN；④用 `any join`/`semi join` 减少结果集；⑤根本上是预聚合（MV）减少 JOIN 数据量。

**③ 踩分点：** 右表进哈希表；grace_hash = 哈希分桶落盘；direct 用排序键点查；partial_merge 只支持 ALL；OOM 切 grace_hash。

---

# 🗜️ 第五部分：压缩与编码

## Q11（⭐⭐）LZ4 和 ZSTD 怎么选？Delta 链式原理？分层压缩策略？

**① 标准答案：**

- **LZ4（默认）**：极速压缩/解压，压缩比中等。适合**热数据、高频扫描列**。
- **ZSTD**：压缩比高（level 1~22），解压比 LZ4 慢。适合**冷数据、历史分区、冗余高列**。Level 过高（15+）压缩极慢，写密集慎用。
- **Delta(N)**：不是压缩算法，是**预处理变换**，存相邻值差值。适合**连续/单调**列（时间戳、自增 ID）。链式 `Delta(N).ZSTD`：Delta 让差值小且重复高，后续通用压缩比大幅提升。

**生产分层策略：**
1. 热分区（近 7 天）用 `LZ4`。
2. 冷分区用 `TTL ... RECOMPRESS CODEC(ZSTD(1))` 自动重压。
3. 时序列加 `Delta(8).ZSTD`。
4. 压缩按"列×part×压缩块"工作，数据量太小换 ZSTD 收益有限。
5. 高频扫描列别用高 level ZSTD。

**其他 codec**：`Gorilla`（时序浮点）、`DoubleDelta`（单调小整数差值）、`T64`（整数）、`FPC`（浮点）、`AES_128_GCM_SIV`（加密）。

**② 🔍 面试官追问：**

- **追问 1：压缩块大小（min/max_compress_block_size）设大设小的影响？**
  答：块大 → 压缩比高（更多数据参考）、但解压一次读更多字节（若只需块内部分数据则浪费 IO）。块小 → 压缩比低、随机读友好。CH 默认 64KB~1MB，是个权衡值。`index_granularity_bytes` 与之配合让 granule 和压缩块对齐。

- **追问 2：为什么列存比行存压缩率高？**
  答：同列数据类型一致、值相近（如某列全是小整数或相似字符串），压缩算法能找到更多重复模式。行存里一行混合多类型，压缩率低。这是列存的核心优势之一。

- **追问 3：Gorilla codec 是干什么的？什么时候用？**
  答：Gorilla 是 Facebook 为时序数据设计的浮点压缩（XOR 编码 + 控制位）。对**时间戳 + 浮点值**的时序场景（如监控指标）压缩率和速度都极好。CH 里用 `CODEC(Gorilla)`。

- **追问 4：加密列怎么处理？影响查询吗？**
  答：用 `AES_128_GCM_SIV` codec 加密整列。查询时自动解密，但**加密列无法被索引/跳过索引有效剪枝**（密文无序），且解密有 CPU 开销。只对敏感 PII 列加密。

**③ 踩分点：** 热用 LZ4 冷用 ZSTD；Delta 是预处理非压缩；列存压缩率天生高；codec 可链式组合。

---

# 🔄 第六部分：数据更新、去重与 Mutation

## Q12（⭐⭐⭐）ReplacingMergeTree、CollapsingMergeTree、AggregatingMergeTree 语义、缺陷、适用场景？

**① 标准答案：**

| 引擎 | 合并行为 | 核心缺陷 | 适用 |
|---|---|---|---|
| **ReplacingMergeTree** | 按 ORDER BY 去重，留最新版本（`ver` 列定序） | **不保证查询时无重复**！必须 `FINAL` 或 `argMax()` | upsert、"最后写入胜出"、去重省空间 |
| **CollapsingMergeTree** | `sign` 列（+1/-1）：+1 与 -1 互抵 | 同样不保证查询时已折叠；查询 SQL 要 `sum(sign)` 重写；写入要先 -1 取消旧行再 +1 写新行 | append-only 模拟行级 UPDATE/DELETE |
| **AggregatingMergeTree** | 按 ORDER BY 对 `AggregateFunction` 列合并 | 必须配合 MV + AggregateFunction 类型列 | 预聚合降存储提查询 |

**关键认知**：**所有 MergeTree 变体的特殊合并逻辑都是异步、最终生效的**。查询当下数据可能仍是"未合并"状态。

**② 🔍 面试官追问：**

- **追问 1：ReplacingMergeTree 的 `ver` 列怎么用？**
  答：`ENGINE = ReplacingMergeTree(ver)` 指定版本列（通常是时间戳或自增 ID）。合并时同 ORDER BY 键的多行保留 `ver` 最大的。没指定 ver 则保留任意一行（通常是最后写入的，但不保证）。

- **追问 2：CollapsingMergeTree 如果 -1 行丢了（没写取消行）会怎样？**
  答：数据错误——旧行没被抵消，查询 `sum(sign)` 会算到旧值。所以 CollapsingMergeTree 要求**严格的 +1/-1 配对**，写入逻辑复杂、易出错。生产里更推荐 VersionedCollapsingMergeTree（多一个 ver 列，更健壮）。

- **追问 3：为什么 CH 不直接做行级 UPDATE/DELETE？**
  答：列存 + 不可变 part + 压缩块的设计，行级修改要重写整个压缩块甚至整个 part，代价巨大。CH 选择"append + 异步合并"模型，UPDATE/DELETE 通过 Replacing/Collapsing 引擎或重量级 Mutation 实现。

- **追问 4：查询时用 `FINAL` 有什么代价？**
  答：`FINAL` 会在查询时**强制应用合并逻辑**（相当于运行时合并），CPU 和内存开销大，且某些优化（如并行读 part）受限。大表 + 多 part 时性能很差。生产优先用 `argMax`/`sum(sign)` 在查询里处理，或定期 OPTIMIZE。

- **追问 5：AggregatingMergeTree 必须配 MV 吗？能直接插数据吗？**
  答：可以直接插，但插进去的要是有意义的聚合状态（用 `*State` 函数生成的 `AggregateFunction` 类型值）。通常配合 MV：MV 把源表的明细聚合后（`uniqState` 等）写入 AggregatingMergeTree 目标表，目标表合并时进一步聚合状态。这是 CH 预聚合的标准范式。

**③ 踩分点：** 三引擎合并都异步最终生效；FINAL 性能差；Collapsing 要求严格配对；Aggregating 配 MV + State 函数。

---

## Q13（⭐⭐⭐）什么是 Mutation？与轻量级 ALTER 区别？为什么生产要尽量避免？

**① 标准答案：**

- **轻量级 ALTER**（仅改元数据，秒级）：`ADD COLUMN`、`RENAME COLUMN`、改 settings、改注释、部分 `MODIFY COLUMN`。
- **重量级 Mutation**（重写 part）：`ALTER TABLE ... UPDATE`、`DELETE`、`MODIFY COLUMN`（改类型/codec）、`MATERIALIZE COLUMN/INDEX`、`CLEAR COLUMN/INDEX`（需重写时）。

**Mutation 内部执行模型：**
1. 进 `system.mutations` 队列，每表一个递增 mutation_id。
2. **重写整个 part**（非原地改行）：复用 merge 机制，读 part 全部行 → 应用谓词 → 写新 part。
3. 同表多 mutation 可**批处理合并成一次扫描**，降 I/O。
4. 复制表 mutation 写入 Keeper 日志，各副本独立但**确定性地**回放（幂等）。
5. mutation_id 记录"哪些 part 需 mutate"，**新插入 part 不受影响**。
6. 与 merge **争抢** `background_pool_size`，被节流。

**为什么生产要避免：** 重写 part → I/O+CPU 极重；大表 mutation 可能跑几小时甚至几天，卡住后续 mutation 队列；不可中断后回滚。

**替代方案：** upsert → ReplacingMergeTree；状态更新 → CollapsingMergeTree；小规模删除 → 轻量级 DELETE（`DELETE FROM ... WHERE`，新版本）；大批删除 → 删分区 + 重插。

**监控：**
```sql
SELECT database, table, mutation_id, command, is_done, parts_to_do
FROM system.mutations WHERE is_done = 0;
KILL MUTATION ON db.table WHERE mutation_id = '...';
```

**② 🔍 面试官追问：**

- **追问 1：轻量级 DELETE 和 ALTER TABLE DELETE 有什么区别？**
  答：轻量级 DELETE（`DELETE FROM t WHERE ...`）是较新特性，标记删除（tombstone），查询时跳过，物理清理靠后续 merge，更快更轻。ALTER TABLE DELETE 是重量级 Mutation，立即重写 part。日常小删用前者，大批删用后者或删分区。

- **追问 2：`ALTER TABLE ADD COLUMN` 真的不重写数据吗？**
  答：是的，只改元数据（columns.txt）。新列对老 part 表现为默认值（DEFAULT），查询时按需填充。但如果老列 `MODIFY COLUMN` 改类型且不兼容，会触发 mutation 重写。

- **追问 3：mutation 卡住几天了怎么办？**
  答：①`KILL MUTATION` 终止；②查 `system.mutations` 看 `parts_to_do` 剩余量；③查 `system.replication_queue`（复制表）看是否卡在副本同步；④若 part 太多，可能是 merge 跟不上，先解决 merge 积压；⑤极端情况删表重建。

- **追问 4：mutations_max_threads 和 background_pool_size 怎么调？**
  答：`background_pool_size`（默认 16）是 merge+mutation 共享池。mutation 太多会挤占 merge。`mutations_max_threads` 限制单 mutation 并行度。生产建议根据机器 IO/CPU 调大 background_pool（如 32~64），但别超 IO 承载力，否则反而互相争抢变慢。

**③ 踩分点：** Mutation 重写 part 非原地改；批处理降 I/O；与 merge 争资源；优先用 Replacing/Collapsing/分区删代替。

---

## Q14（⭐⭐）OPTIMIZE FINAL 是什么？为什么是双刃剑？

**① 标准答案：** `OPTIMIZE TABLE t FINAL` 强制把所有 part 合并成一个 part 并应用特殊合并逻辑（去重/折叠），使查询当下看到合并后结果。

**双刃剑：** 强制全量合并 → 巨大 I/O+CPU+临时空间，可能阻塞正常 merge；大表可能跑数小时，产生超大 part 违背"渐进合并"；只是"一次性补救"，新数据进来又回到未合并状态。

**正确姿势：** 只在数据导入完成后做一次性 OPTIMIZE，或在小表/特定分区上 `OPTIMIZE PARTITION ... FINAL`；日常去重靠查询层（argMax / FINAL）。

**② 🔍 面试官追问：**

- **追问 1：OPTIMIZE 不加 FINAL 行不行？**
  答：`OPTIMIZE TABLE t`（无 FINAL）只触发常规合并（把小 part 合成大 part），不强制应用 Replacing/Collapsing 的特殊逻辑。所以去重效果不保证。FINAL 才强制应用。

- **追问 2：OPTIMIZE 后产生的超大 part 会不会有问题？**
  答：会。一个超大 part（如 150GB）后续任何 mutation/合并都极慢，且单 part 无法并行读（part 是并行读的最小单位之一）。所以 OPTIMIZE FINAL 不该常用。

- **追问 3：有什么轻量替代方案？**
  答：①定期 `OPTIMIZE PARTITION ... FINAL` 只处理热点分区；②查询层用 `argMax`/`sum(sign)`；③靠自然 merge 渐进处理（监控 part 数）。

**③ 踩分点：** FINAL 强制合并+应用逻辑；代价巨大；只在导入收尾或小分区用。

---

# 💾 第七部分：内存管理与生产稳定性

## Q15（⭐⭐）ClickHouse 内存控制机制？OOM 常见原因和规避？

**① 标准答案：**

**控制机制：**
- `max_memory_usage`：单查询内存上限（默认 10GB）。
- `max_memory_usage_for_all_queries`：全局上限。
- `max_bytes_before_external_group_by` / `max_bytes_before_external_sort`：超阈值**溢写磁盘**。
- 聚合用专门哈希表（30+ 变体，按类型/基数选最优）。

**OOM 常见原因：** 大 GROUP BY 没开外部聚合；大表 JOIN 右表进内存；`SELECT *`；`max_rows_to_group_by` 触发 throw；高并发打满全局上限。

**规避：** 开 `max_bytes_before_external_group_by`（建议 = max_memory_usage 的 0.5~0.7）；大 JOIN 用 grace_hash/partial_merge；用 AggregateFunction + MV 预聚合；限制 `max_threads`；用 `system.query_log`/`system.processes` 排查。

**② 🔍 面试官追问：**

- **追问 1：外部聚合（external group by）原理是什么？**
  答：GROUP BY 超过内存阈值时，把哈希表按分组键哈希**分桶落盘**，然后逐桶读回做最终聚合。类似 grace hash 的思想。代价是磁盘 IO + 多轮处理。开启：`max_bytes_before_external_group_by = N`。

- **追问 2：为什么建议 external_group_by 阈值 = max_memory_usage 的 0.5~0.7？**
  答：留 30%~50% 给后续算子（排序、JOIN、结果集）。如果阈值 = max_memory_usage，聚合用满后下游算子没内存就 OOM。

- **追问 3：`max_threads` 和内存什么关系？**
  答：很多算子（聚合、排序）是**每线程独立数据结构**。`max_threads=16` 时，单个聚合哈希表可能是 16 份。线程越多内存占用越大。内存紧张时降 `max_threads` 比降阈值更有效。

- **追问 4：怎么定位是哪个查询吃内存？**
  答：①`system.processes`（实时）看 `memory_usage`；②`system.query_log`（历史）看峰值 `memory_usage` 和 `ProfileEvents`；③`system.query_thread_log` 看每线程；④`top`/`htop` 看 CH 进程。设 `log_query_threads=1` 开线程级日志。

**③ 踩分点：** 外部聚合分桶落盘；阈值留余量给下游；max_threads 影响内存；system 表排查。

---

# 🏗️ 第八部分：物化视图与生产设计

## Q16（⭐⭐）ClickHouse 物化视图和 PostgreSQL MV 的本质区别？AggregatingMergeTree + MV 经典模式？

**① 标准答案：**

- **PostgreSQL MV**：存查询快照，需 `REFRESH` 刷新，全量重算。
- **ClickHouse MV**：本质是**插入触发器（insert trigger）**。每次源表插入，CH 自动把新数据块喂给 MV 的 SELECT，结果写入 MV 的目标表（TO table）。**增量、实时、自动**，不重算历史。

**经典模式：AggregatingMergeTree + MV：**
1. 源表（明细）：`MergeTree`。
2. MV：`CREATE MATERIALIZED VIEW mv TO agg_table AS SELECT dim, uniqState(uid) AS uv, sumState(amount) AS amt FROM source GROUP BY dim`。
3. 目标表：`AggregatingMergeTree`，列类型 `AggregateFunction(uniq, ...)`。
4. 查询：`SELECT dim, uniqMerge(uv), sumMerge(amt) FROM agg_table GROUP BY dim`。

**陷阱：** MV 只对新插入触发，不回填历史（除非 POPULATE，有竞争风险）；删源表不自动删 MV；多 MV 让插入变慢；MV 的 SELECT 不能跨节点 JOIN。

**② 🔍 面试官追问：**

- **追问 1：POPULATE 是什么？为什么不推荐？**
  答：`CREATE MATERIALIZED VIEW ... POPULATE` 在建 MV 时把现有源表数据回填一遍。**但创建期间新插入的数据可能被漏掉或重复**（竞争条件）。生产推荐：先建 MV（空），再手动用 INSERT ... SELECT 补历史，期间停写或用版本号去重。

- **追问 2：MV 的 SELECT 里能 JOIN 吗？**
  答：技术上能，但**跨节点 JOIN 在分布式 MV 上有问题**（同 GLOBAL JOIN 陷阱）。生产推荐 MV 只做单表聚合，维度 JOIN 留在查询层，或维度表复制到每分片。

- **追问 3：一个源表挂多个 MV 会怎样？**
  答：每次插入，所有 MV 都执行一遍 SELECT，**插入延迟线性增加**。通常 3~5 个 MV 是上限，再多要考虑用一张宽表 + 查询层聚合代替。

- **追问 4：MV 目标表用 AggregatingMergeTree 还是普通的 MergeTree？**
  答：预聚合用 AggregatingMergeTree（列是 AggregateFunction 类型，合并时自动聚合状态）。如果 MV 只是筛选/投影（不聚合），用普通 MergeTree 即可。

**③ 踩分点：** CH MV = 插入触发器（增量实时）；不回填历史；POPULATE 有竞争；多 MV 拖慢插入。

---

## Q17（⭐⭐）分区和排序键哪个加速查询？分区怎么设计？

**① 标准答案（官方原话）：**

**"Partitioning does not speed up queries (in contrast to the ORDER BY expression)."** —— **排序键加速查询，分区主要服务于数据生命周期管理（TTL 删/移动）**。分区提供**分区剪枝**（跳过整个分区目录），但不像排序键在分区内细粒度 granule 剪枝。

**生产分区设计：**
1. **粒度：按月（`toYYYYMM(dt)`）最常见**，官方建议"通常不需要比月更细"。
2. **目的**：按时间 TTL 删旧、冷热分离（TTL MOVE TO disk）、`DROP PARTITION` 快速删。
3. **反模式**：按天分区（分区爆炸）、按客户 ID 分区（不均）、把高基数列分区（应放 ORDER BY 前列）。
4. **part 数红线**：单分区建议不超 ~1000 part。

**② 🔍 面试官追问：**

- **追问 1：为什么分区不加速查询，却能剪枝？这不矛盾吗？**
  答：剪枝是"跳过整个分区"（粗粒度，类似跳过 N 个 part），减少读的 part 数。但单个分区内仍要靠排序键做 granule 剪枝才能细粒度加速。分区加速的是"删/移动数据"和"减少 part 数"，不是查询内的行过滤。

- **追问 2：按天分区到底有什么问题？**
  答：①part 数爆炸（每天 N 个 part，一年 365N）；②Keeper 元数据膨胀（复制表每个 part 一堆 znode）；③merge 跟不上；④查询 planning 慢（要枚举大量 part）。CH 偏好**少量大 part**，不是大量小分区。

- **追问 3：分区键和排序键可以重叠吗？**
  答：可以且常见，如 `PARTITION BY toYYYYMM(dt) ORDER BY (dt, user_id)`。dt 在分区和排序键都出现，分区做粗剪枝，排序键做细剪枝，协同高效。

- **追问 4：怎么删历史数据最省资源？**
  答：`ALTER TABLE ... DROP PARTITION` 或配 TTL 自动删。删分区是**元数据操作**（删整个目录），不重写 part，极快。比 mutation DELETE 快几个数量级。这是分区设计的核心价值。

**③ 踩分点：** 排序键加速查询，分区服务生命周期；按月最常见；删分区是元数据操作极快；part 数红线。

---

# 🆕 新增进阶模块

---

# 📥 模块九：数据摄入与异步写入

## Q18（⭐⭐⭐）Kafka 引擎表、Buffer 引擎、Async Insert 各自的原理与适用？生产摄入架构怎么选？

**① 标准答案：**

**三种摄入机制：**

| 机制 | 原理 | 优点 | 缺点 |
|---|---|---|---|
| **Kafka 引擎表** | CH 直接消费 Kafka，配合 MV 把数据落到 MergeTree | 端到端简单、自动消费 | CH 挂了会丢未消费数据；消费速率受 CH 影响；故障恢复复杂 |
| **Buffer 引擎** | 写入先到内存 Buffer 表，达阈值批量刷到目标表 | 攒批小写入，减少 part 数 | 数据在内存，CH 崩溃丢；Buffer 表本身不复制 |
| **Async Insert** | 客户端小批量 INSERT，服务端按时间/数量攒批后落盘 | 客户端无需攒批，省心；服务端攒批 | 攒批期间数据在服务端内存；有延迟（`async_insert_max_data_size`/`wait_for_async_insert`） |

**生产摄入架构推荐：**
1. **首选**：Kafka（或 Pulsar/RocketMQ）+ 独立消费者（如 vector、logstash、自研）攒批 → 批量写 CH 本地表。CH 不直接消费，解耦。
2. **次选**：CH Kafka 引擎表 + MV 落地（简单场景）。
3. **高频小写**：客户端用 Async Insert（`async_insert=1, wait_for_async_insert=0`）。
4. **核心原则**：**批量写入**，每批 ≥ 1 万行，每秒 ≤ 1~2 次 INSERT，避免 "Too many parts"。

**② 🔍 面试官追问：**

- **追问 1：为什么会 "Too many parts"？**
  答：每次 INSERT 产生一个 part。若写入频率太高（如每秒 1000 次小写），part 数暴涨超过 `parts_to_throw_insert`（默认 3000），CH 直接拒绝写入保护自己。根本解法是攒批写入。

- **追问 2：Kafka 引擎表消费失败怎么保证不丢数据？**
  答：Kafka 引擎表消费后通过 MV 落地，offset 提交时机要谨慎（落地成功才提交）。但 CH 本身不是消息系统的可靠消费者，生产更推荐外部消费者（vector 等）+ 手动 offset 管理 + 死信队列。

- **追问 3：Async Insert 的攒批发生在哪？客户端还是服务端？**
  答：服务端。客户端发小 INSERT，服务端按 `async_insert_max_data_size`（默认 100000 字节）或 `async_insert_busy_timeout_ms` 攒批，攒够后一次落盘。多个客户端的写入会被服务端合并攒批。

- **追问 4：Buffer 引擎为什么不能用于关键数据？**
  答：Buffer 表数据在内存，CH 崩溃/重启会丢；且 Buffer 表**不参与复制**（不是 Replicated），副本间不一致。只适合可容忍丢失的临时缓冲。

**③ 踩分点：** 攒批是摄入核心；Kafka 解耦优于 CH 自消费；Async Insert 服务端攒批；Buffer 不复制有丢数据风险。

---

## Q19（⭐⭐⭐）什么是 Projection（投影）？和跳过索引、物化视图有什么区别？什么时候用？

**① 标准答案：**

**Projection** 是在**同一张表内**存储一份按不同排序键排好序的数据副本（可带聚合）。查询时优化器自动选择用 Projection 还是原表。

**与其他机制对比：**

| 机制 | 本质 | 数据冗余 | 自动选择 | 适用 |
|---|---|---|---|---|
| **Projection** | 同表内不同排序键的副本 | 是（写在同表 part 子目录） | 优化器自动 | 排序键冲突，多维度查询 |
| **跳过索引** | granule 级元数据（minmax/bloom） | 几乎不冗余 | 自动 | 非排序键列的额外剪枝 |
| **物化视图** | 独立表 + 插入触发器 | 是（独立表） | 需查 MV 表名 | 预聚合、行过滤、完全不同的查询模式 |

**Projection 适用场景：**
- 主排序键是 `(user_id, event_time)`，但常按 `device_id` 查询 → 建 `ORDER BY (device_id, event_time)` 的 Projection。
- 可带聚合：`SELECT user_id, sum(amount) GROUP BY user_id` 的 Projection 加速聚合查询。

**代价：** 存储翻倍；写入变慢（每次插入要写多份）；合并变慢。

**② 🔍 面试官追问：**

- **追问 1：Projection 和建一张独立的排序表有什么区别？**
  答：Projection 的最大优势是**优化器自动透明选择**——查询原表时优化器自动判断用哪个 Projection 最优，业务 SQL 不用改。独立排序表要业务显式查那张表。

- **追问 2：Projection 会自动更新吗？**
  答：会。Projection 是 part 的子目录，随主表插入和合并自动维护，不需要单独刷新。这是它比独立 MV 表省心的地方。

- **追问 3：Projection 太多会怎样？**
  答：①写入放大（每个 Projection 一份数据）；②合并变慢（每个 Projection 都要重排）；③part 变大。通常 1~2 个 Projection 是上限。

- **追问 4：什么时候用 Projection，什么时候用 MV？**
  答：①只想换排序键加速查询、SQL 不想改 → Projection；②要做预聚合（减少行数）或行过滤 → MV（独立表更灵活）；③既要换排序又要聚合 → Projection 的聚合版本。Projection 更透明但灵活性低，MV 更灵活但要改 SQL。

**③ 踩分点：** Projection 同表内透明自动选择；代价是写入放大；与跳过索引/MV 的本质区别是"自动 vs 显式"。

---

## Q20（⭐⭐）Dictionary（字典）是什么？解决什么问题？

**① 标准答案：** Dictionary 是 CH 的**内存/本地缓存维度表**机制。把维度数据（如用户信息、商品信息）加载到 CH 内存（或本地文件），用于：
1. **JOIN 加速**：`dictGet` 函数直接查字典，无需 JOIN。
2. **属性补充**：查询时实时 enrich 维度属性。

**字典类型（layout）：**
- `FLAT`：数组下标存（key 必须是 < 5 亿的整数）。
- `HASHED`：哈希表。
- `CACHE`：部分缓存（数据源是 MySQL/HTTP 等）。
- `RANGE_HASHED`：带范围查询。
- `COMPLEX_KEY_HASHED`：复合键。

**数据源：** ClickHouse 表、MySQL、HTTP、文件、MongoDB 等。

**② 🔍 面试官追问：**

- **追问 1：dictGet 比 JOIN 快在哪？**
  答：字典常驻内存，`dictGet` 是函数级直接哈希查找，没有 JOIN 的算子开销、哈希表构建、数据传输。对"大事实表 JOIN 小维度表"场景，dictGet 比分布式 JOIN 快一个数量级。

- **追问 2：字典数据怎么更新？**
  答：配 `LIFETIME` 定期刷新，或用 `RELOAD DICTIONARY` 手动刷。也可配 `CLICKHOUSE` 源 + 物化视图实现准实时。大字典全量刷新有开销。

- **追问 3：字典能存多大？**
  答：取决于内存。FLAT 最大 5 亿 key（整数）。HASHED 理论无上限但受内存约束。太大字典用 CACHE（部分缓存）或干脆复制成 Replicated 表做本地 JOIN。

**③ 踩分点：** 字典 = 内存维度表；dictGet 替代 JOIN；按 layout 选；定期刷新。

---

## Q21（⭐⭐）Query Cache 是什么？有什么陷阱？

**① 标准答案：** Query Cache 缓存 SELECT 查询的结果（行+块），相同查询再次来时直接返回。配 `enable_query_cache`、`query_cache_max_size_in_bytes` 等。

**陷阱：**
1. **结果可能过期**：表数据变了，缓存不自动失效（除非配 `query_cache_min_query_runs` 等条件）。对实时性要求高的场景慎用。
2. **缓存命中率**：参数化查询（不同 WHERE 值）缓存不命中。适合固定报表查询。
3. **内存占用**：大结果集缓存吃内存。
4. **非确定性函数**：`now()`、`rand()` 等结果不缓存或缓存错误。

**② 🔍 面试官追问：**

- **追问 1：什么场景适合开 Query Cache？**
  答：①固定报表（BI dashboard 每天跑相同查询）；②数据更新频率低（T+1 数据）；③查询结果不大。不适合实时分析、参数化 ad-hoc 查询。

- **追问 2：缓存什么时候失效？**
  答：CH 对缓存条目设 TTL 和大小上限，LRU 淘汰。表数据变更**不会自动失效**对应缓存（除非配 `query_cache_min_query_duration`、`query_cache_min_query_runs` 等过滤条件 + 手动清理）。所以实时场景要小心。

**③ 踩分点：** 适合固定报表；数据变更不自动失效；非确定性函数要排除。

---

# ☁️ 模块十：存储分离与冷热分层

## Q22（⭐⭐⭐）ClickHouse 怎么做冷热分层存储（Tiered Storage）？S3 存储分离架构？

**① 标准答案：**

**Tiered Storage（冷热分层）：**
1. 在 `config.xml` 配置多个 **storage policy**（存储策略），每个策略含多个 **volume**，每个 volume 含多个 **disk**。
2. 例：`hot` volume（本地 NVMe disk）→ `cold` volume（S3 disk）。
3. 表配 `TTL dt + INTERVAL 7 DAY TO VOLUME 'cold'`，老数据自动从热盘迁到 S3。
4. `MOVE PARTITION ... TO DISK` 手动迁移。

**S3 存储分离架构：**
- CH 支持 `s3` disk 类型，part 直接存 S3 对象。
- 适合存算分离：计算节点无状态、扩缩容快；S3 海量廉价。
- 代价：S3 延迟比本地盘高（GET 延迟几十~几百 ms），查询热数据建议本地缓存（CH 有 `s3_cache`）。
- 写入：part 仍先在本地临时目录，合并后上传 S3。

**② 🔍 面试官追问：**

- **追问 1：S3 上查询性能怎样？怎么优化？**
  答：裸 S3 查询慢（网络+对象存储延迟）。优化：①开 `s3_cache`（本地盘缓存热块）；②用更激进的压缩减少传输；③合理 `index_granularity` 减少读取量；④热数据仍放本地 NVMe，冷数据才 S3（分层）。

- **追问 2：TTL MOVE 到冷盘是同步还是异步？**
  答：异步。后台 merge/move 线程执行，按 `move_factor`（默认 0.1）控制。迁移期间数据可能在两个 volume 都存在（短暂）。监控 `system.parts` 的 `disk_name`。

- **追问 3：S3 上的 part 和本地 part 有什么区别？**
  答：逻辑上一样（都是 part 目录 + 列文件），物理上 part 的列文件是 S3 对象。CH 用 S3 API 读写。元数据（primary.idx、mrk2）也存 S3，但会缓存到本地加速查询。

- **追问 4：纯 S3 存储有什么风险？**
  答：①S3 宕机或网络抖动导致查询失败；②一致性模型（S3 强一致但跨操作有延迟）；③List 操作慢影响 part 枚举；④成本：请求次数多了 S3 费用高。生产通常本地热盘 + S3 冷盘混合。

**③ 踩分点：** storage policy/volume/disk 三层；TTL MOVE 异步；S3 适合冷数据 + 缓存；热数据仍放本地。

---

## Q23（⭐⭐）TTL 除了删数据还能做什么？写一个完整的 TTL 配置示例。

**① 标准答案：** TTL 不仅能删数据，还能：①**移动**到其他 volume/disk（冷热分层）；②**重组件**（`RECOMPRESS` 换压缩算法）；③**删列**（`DELETE` 特定列）。

**完整示例：**
```sql
CREATE TABLE events (
    dt DateTime,
    user_id UInt64,
    amount Decimal(10,2),
    raw_log String
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(dt)
ORDER BY (dt, user_id)
TTL
    dt + INTERVAL 30 DAY RECOMPRESS CODEC(ZSTD(5)),   -- 30天后换高压缩
    dt + INTERVAL 90 DAY TO VOLUME 'cold',            -- 90天后迁冷盘
    raw_log + INTERVAL 180 DAY DELETE,                -- 180天后删raw_log列
    dt + INTERVAL 365 DAY DELETE                      -- 365天后删整行
SETTINGS storage_policy = 'hot_cold';
```

**② 🔍 面试官追问：**

- **追问 1：TTL 是怎么执行的？实时吗？**
  答：异步。后台 TTL merge 线程周期性扫描，对过期数据应用 TTL 规则（重压缩/迁移/删除）。不是实时，有延迟（受 `merge_with_ttl_timeout` 控制）。急用可 `ALTER TABLE ... MATERIALIZE TTL`。

- **追问 2：TTL 和分区删除哪个省资源？**
  答：分区删除（DROP PARTITION）是元数据操作，极快。TTL DELETE 是行级（实际重写 part），慢。所以按时间删整批数据用分区 + DROP PARTITION，精细删除才用 TTL。

- **追问 3：TTL RECOMPRESS 什么时候触发？**
  答：数据到期时，下次 merge 会对这些 part 用新 codec 重新压缩。本质是 merge 时应用 TTL 规则，不是独立操作。

**③ 踩分点：** TTL 四功能（删/移/压/删列）；异步 merge 执行；DROP PARTITION 比 TTL DELETE 快。

---

# 🔑 模块十一：分片键设计与水平扩展

## Q24（⭐⭐⭐）分片键（sharding key）怎么设计？分片数怎么定？扩容怎么处理？

**① 标准答案：**

**分片键设计原则：**
1. **均匀分布**：避免数据倾斜。常用 `rand()`、`cityHash64(col)`、`intHash64(col)`。
2. **查询局部性**：让常一起查的数据落同一分片，减少跨分片查询。如按 `tenant_id` 分片，单租户查询不跨分片。
3. **与排序键协同**：分片键决定数据在哪个节点，排序键决定节点内顺序。

**分片数：** 不是越多越好。每个分片是一个独立节点（或副本组）。分片多 → ①元数据多；②跨分片查询网络开销；③每分片数据少，单分片查询无法利用多核。通常按数据量和查询并发算：单分片 1~30TB 为宜。

**扩容（加分片）：**
- **痛点**：CH 的分片键一旦定下，已有数据不会自动迁移。加新分片后，老数据还在老分片，新数据才进新分片。
- **方案**：①重新分片（建新集群 + 双写 + 迁移 + 切流）；②用一致性哈希（CH 不原生支持，需自研）；③预先规划足量分片，靠副本扩容而非加分片。

**② 🔍 面试官追问：**

- **追问 1：分片键用 `rand()` 有什么问题？**
  答：数据均匀分布好，但**查询无局部性**——任何按业务维度的查询都要扫所有分片。适合纯随机分析场景，不适合多租户。

- **追问 2：按 `tenant_id` 分片，大租户怎么办？**
  答：大租户数据集中在一个分片，造成热点。解法：①大租户单独分片（白名单路由）；②大租户内部再按子键二次分片；③监控各分片数据量，手动平衡。

- **追问 3：Distributed 表的分片配置在哪？**
  答：在 `config.xml` 的 `<remote_servers>` 或用 `cluster` 配置（XML 或 `system.clusters`）。Distributed 表 DDL 里 `ENGINE = Distributed(cluster, db, table, sharding_key)`，`sharding_key` 是分片表达式。

- **追问 4：跨分片 JOIN 为什么慢？**
  答：要么把右表广播到所有分片（GLOBAL），要么把数据拉到 initiator 再 JOIN。两者都有大量网络传输和内存开销。所以维度表复制到每分片做本地 JOIN 是最优解。

**③ 踩分点：** 分片键要均匀+局部性；分片数按数据量算；扩容难（不自动迁移）；维度表复制解 JOIN。

---

# 🗄️ 模块十二：备份、迁移与运维

## Q25（⭐⭐）ClickHouse 怎么做备份恢复？有哪些方案？

**① 标准答案：**

**备份方案：**

| 方案 | 原理 | 优缺点 |
|---|---|---|
| **BACKUP/RESTORE 命令** | 原生命令，备份到 File/S3/Azure 等 | 官方推荐；增量支持；灵活 |
| **clickhouse-backup**（第三方） | 快照 part 到 S3 | 社区流行；自动化好 |
| **复制表本身** | ReplicatedMergeTree 多副本即冗余 | 不是备份，防单点；误删不防 |
| **part 文件级拷贝** | 直接 cp part 目录 | 简单但易错；不保证一致 |
| **ALTER ... FREEZE** | 对 part 做硬链接快照 | 节省空间；配合 rsync 上传 |

**② 🔍 面试官追问：**

- **追问 1：复制能代替备份吗？**
  答：不能。复制只防单机故障，**误删/误操作（如 DROP TABLE、错误的 mutation）会被同步到所有副本**，复制救不了。备份是独立于集群的离线副本。

- **追问 2：BACKUP 是在线备份吗？会锁表吗？**
  答：在线，不锁表。BACKUP 基于某一时刻的 part 快照，备份期间写入正常进行（新 part 不进备份）。一致性是 part 级别的，不是事务级。

- **追问 3：怎么备份海量数据省空间？**
  答：①增量备份（只备变化 part）；②备份到 S3 + ZSTD 压缩；③用 FREEZE 做硬链接快照（CP 协议），省去复制空间；④按分区/表分批备份。

- **追问 4：恢复时表已存在怎么办？**
  答：RESTORE 要么到新表名，要么先 DROP 再 RESTORE。CH 备份是 part 级，不像 MySQL 有 binlog point-in-time 恢复。生产要配合定期全备 + 增量。

**③ 踩分点：** 复制不能代替备份；BACKUP 在线不锁表；FREEZE 硬链接省空间；增量备份省存储。

---

## Q26（⭐⭐⭐）线上查询突慢，系统化排查路径？

**① 标准答案（结合 system 表）：**

1. **看 query_log**：`system.query_log` 找该查询的 `event_time`、`query_duration_ms`、`read_rows`、`read_bytes`、`memory_usage`、`exception`。对比正常 vs 异常时段。
2. **判断是否没走剪枝**：
   - `read_rows` 接近全表 → 排序键/分区没命中 → 检查 WHERE 是否用了排序键前列、类型是否匹配、是否函数包裹导致失效（如 `WHERE toDate(dt) = '2026-01-01'` 不如 `WHERE dt >= '2026-01-01' AND dt < '2026-01-02'`）。
   - `read_rows` 少但慢 → 高并发争抢 CPU/IO，或外部聚合溢写磁盘。
3. **看 part 数**：`system.parts WHERE active` → part 是否暴涨（合并跟不上）。
4. **看 merge/mutation**：`system.merges`、`system.mutations` → 是否有大量 mutation 拖慢 IO。
5. **看内存**：`system.processes` → 是否触顶、是否 external aggregation（看 `ProfileEvents` spill 次数）。
6. **看硬件**：CPU/IO/磁盘延迟/网络（分片间）。
7. **看数据增长**：是否数据量翻倍、基数爆炸（GROUP BY 分组数突增）。
8. **看 plan**：`EXPLAIN` / `EXPLAIN PLAN` 看算子（是否退化全表扫、JOIN 是否选了 hash 还是 merge）。

**② 🔍 面试官追问：**

- **追问 1：`WHERE toDate(dt) = '2026-01-01'` 为什么慢？**
  答：对排序键列用函数（`toDate`）包裹，**破坏了单调性**，CH 无法做二分剪枝，退化为逐 granule 扫描。应改写为范围查询 `WHERE dt >= '2026-01-01' AND dt < '2026-01-02'`，保持列原生单调性。

- **追问 2：怎么看一个查询用了几个线程？**
  答：`system.query_log` 的 `query_id` 关联 `system.query_thread_log`（需 `log_query_threads=1`），看线程数。或查询时设 `send_logs_level='trace'` 看日志。线程数受 `max_threads` 和数据量影响。

- **追问 3：part 数多少算多？**
  答：单表单分区超过几百个 active part 就要警惕，上千就有问题（查询 planning 慢、merge 跟不上）。看 `system.parts WHERE table='x' AND active AND partition='...'`。

- **追问 4：怎么定位慢查询？**
  答：①`system.query_log WHERE query_duration_ms > N ORDER BY ...`；②设 `log_query_duration=1000`（超 1s 记录）；③用 `system.processes`（实时，`query` + `elapsed`）查当前执行中的慢查询；④配 `max_query_size`、`readonly` 等限制。

**③ 踩分点：** query_log 是核心；函数包裹破坏剪枝；part 数红线；EXPLAIN 看执行计划。

---

## Q27（⭐⭐）为什么 ClickHouse "不擅长"点查和高频小写入？

**① 标准答案：**

- **点查不擅长**：稀疏索引最细到 granule（8192 行），单点查也要流式读 8192 行 SIMD 过滤，QPS 上限低（单机几千），远不如 InnoDB B+Tree 点查。
- **高频小写入不擅长**：每次 INSERT 产生一个 part。每秒插 1000 次每次 1 行 → 海量小 part → "Too many parts" 报错、合并跟不上、planning 慢。CH 要求**批量写入**（每批 ≥ 1 万行，每秒 ≤ 1~2 次 INSERT）。

**解决：**
- 点查 → 外部缓存（Redis）、Buffer 引擎、Projection 改排序键。
- 高频小写 → 前端 Kafka/队列攒批，或 Buffer/Kafka 引擎表，或 Async Insert。

**② 🔍 面试官追问：**

- **追问 1：CH 到底适合什么场景？**
  答：**OLAP 分析型负载**——大范围扫描、聚合、GROUP BY、时序分析、日志/事件分析。批量大写入（实时但批量）、少量并发的大查询。不适合 OLTP（点查、事务、高频小写、强一致更新）。

- **追问 2：能不能用 CH 做用户画像的点查（按 user_id 查一行）？**
  答：能但不优。若 user_id 是排序键首位，点查靠二分稀疏索引 + granule 过滤，毫秒级，但 QPS 低（单核几百~几千）。高 QPS 场景（如推荐系统实时查画像）建议 Redis 缓存 + CH 做离线计算。

- **追问 3：实时写入（流式）怎么平衡？**
  答：①Kafka 攒批（推荐，秒级延迟）；②Async Insert（客户端无需攒批，服务端攒批）；③Buffer 引擎（小心丢数据）。核心是控制 INSERT 频率，不是控制数据延迟——秒级延迟通常可接受。

**③ 踩分点：** CH = OLAP；稀疏索引点查弱；批量写入是铁律；流式写靠攒批。

---

# 🧠 高频"陷阱题"快查表（面试官最爱挖坑）

| 问题 | 关键答案 |
|---|---|
| ReplacingMergeTree 查询时数据一定去重了吗？ | **没有**，必须 FINAL 或 argMax。 |
| 分区能加速查询吗？ | **不能**（只剪枝+生命周期），加速靠 ORDER BY。 |
| 主索引能精确定位单行吗？ | **不能**，最细到 granule（8192 行）。 |
| bloom_filter 跳过索引能优化 `!=` 吗？ | **不能**，布隆只能证伪。 |
| 写数据写本地表还是分布式表？ | **本地表**。 |
| ClickHouse 支持事务吗？ | 不支持 ACID，仅原子批量写入。 |
| Mutation 能回滚吗？ | 不能，只能 KILL，已写 part 不撤。 |
| 物化视图会回填历史吗？ | **不会**（除非 POPULATE，有竞争）。 |
| JOIN 默认哪张表进内存？ | **右表**，小表放右边。 |
| ZooKeeper 和 Keeper 能混用吗？ | 协议兼容可替换，但**同集群别混用**。 |
| count distinct 跨分片怎么算？ | 用 `uniqState`/`uniqMerge`，别直接 count distinct。 |
| 复制是强一致吗？ | **最终一致**，异步回放。 |
| `WHERE toDate(dt)=x` 为什么慢？ | 函数包裹破坏单调性，无法剪枝。 |
| Projection 会自动更新吗？ | **会**，随 part 插入合并自动维护。 |
| TTL 是实时执行吗？ | **异步**，merge 时执行。 |
| Async Insert 攒批在哪？ | **服务端**。 |
| 复制能代替备份吗？ | **不能**，误操作会同步到所有副本。 |
| S3 存储查询快吗？ | 慢，要配 `s3_cache`，热数据放本地。 |

---

# 🏗️ 系统设计大题（Part 13 · 高级架构师岗）

> 这类题没有唯一标准答案，考察的是**容量估算、架构权衡、踩坑经验、追问深度**。每题按"需求拆解 → 架构图 → 关键决策 → 容量估算 → 追问陷阱"组织。

---

## 🎯 大题一：设计一个千万 QPS 的电商实时数仓（最常考）

### 需求描述
电商埋点系统：用户行为事件（点击、加购、下单、支付），峰值写入 **1000 万 QPS**，需支持：
1. **实时大屏**：GMV、UV、转化率，秒级刷新。
2. **BI 自助分析**：运营按时间/品类/渠道多维分析，查询 P95 < 5s。
3. **用户行为分析**：单用户行为轨迹查询，P95 < 500ms。
4. **数据保留**：明细 90 天，聚合 3 年。

### ① 需求拆解与容量估算（先算清楚！）

```
写入：1000 万 QPS（峰值）→ 平均约 300 万 QPS
日均事件：300万 × 86400 ≈ 2600 亿条/天
单条大小：约 500 字节（含 30+ 字段，user_id/event_type/item_id/amount/ts/...）
日均原始数据：2600亿 × 500B ≈ 130 TB/天（未压缩）
列存+LZ4 压缩比约 7~10x → 压缩后 ≈ 15 TB/天
90 天明细：≈ 1.3 PB（压缩后）
3 年聚合：聚合后约 1/1000 → 约 15 TB

⚠ 关键认知：CH 单节点扛不住 1000 万 QPS 写入，必须【分片 + 攒批】
```

### ② 整体架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                          数据源（埋点 SDK）                          │
│              App/Web/小程序 → 上报事件（1000万 QPS）                  │
└──────────────────────────────┬──────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    接入层：负载均衡 + 网关                            │
│            Nginx/LB → 网关集群（鉴权、限流、采样）                    │
└──────────────────────────────┬──────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│               缓冲层：Kafka（或 Pulsar/RocketMQ）                     │
│      Topic: events，按 user_id 分区（如 100+ 分区）                   │
│      作用：削峰、解耦、重放、保证不丢                                 │
└──────────────────────────────┬──────────────────────────────────────┘
                               ▼  （消费者攒批）
┌─────────────────────────────────────────────────────────────────────┐
│           摄入层：vector / logstash / 自研消费者                      │
│      从 Kafka 拉取，攒批（每批 5~10 万行 / 每 2 秒）                  │
│      → 批量 INSERT 到 CH 本地表（每分片一个消费者组）                 │
└──────────────────────────────┬──────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 存储层：ClickHouse 集群                              │
│   Distributed 表（逻辑视图）                                         │
│        ├─ 分片1 (ReplicatedMergeTree) ── 2 副本                      │
│        ├─ 分片2 (ReplicatedMergeTree) ── 2 副本                      │
│        ├─ ... × N 分片                                               │
│        + AggregatingMergeTree 预聚合目标表（经物化视图）              │
│   Keeper 集群（3 节点）协调复制                                       │
└──────────────────────────────┬──────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│             查询层：BI 工具 / 大屏 / 应用 API                        │
│   Superset/Metabase（BI）｜Grafana（大屏）｜自研 API（行为分析）       │
│   + Query Cache（固定报表）+ Redis（高频点查缓存）                    │
└─────────────────────────────────────────────────────────────────────┘
```

### ③ CH 集群规划（关键决策）

**分片数怎么定？**
```
写入吞吐目标：1000万 QPS ÷ 每分片写入能力
单分片写入能力：攒批后约 5~10 万 行/秒（INSERT），即 ~30万行/秒上限
理论最少分片：1000万 / 30万 ≈ 34 个分片
冗余 + 峰值缓冲：取 2 倍 → 约 60~80 个分片

每个分片：1 主 + 1 备副本（ReplicatedMergeTree），高可用
单分片规格：32 核 / 128GB / 4TB NVMe（热数据）
冷数据：S3（TTL 30 天后迁移）
```

**为什么不全用副本抗写？** 副本是异步复制，写入只到一个副本再同步，不提升写吞吐。**写吞吐靠分片水平扩展**。

### ④ 表设计（核心）

```sql
-- 明细表（分布式，按 user_id 哈希分片）
CREATE TABLE events_local ON CLUSTER cluster (
    event_id     String CODEC(ZSTD(3)),
    user_id      UInt64,
    event_type   LowCardinality(String),   -- 低基数用 LowCardinality 省内存
    item_id      UInt64,
    category     LowCardinality(String),
    channel      LowCardinality(String),
    amount       Decimal(10,2),
    dt           DateTime,
    date         Date MATERIALIZED toDate(dt),
    raw          String DEFAULT ''          -- 扩展字段
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/events', '{replica}')
PARTITION BY toYYYYMM(dt)                    -- 按月分区（删历史 + 剪枝）
ORDER BY (user_id, dt)                       -- 排序键：user_id 在前（行为查询）
                                              --         dt 在后（时间范围）
TTL dt + INTERVAL 30 DAY TO VOLUME 'cold',   -- 30天迁 S3
    dt + INTERVAL 90 DAY DELETE              -- 90天删除
SETTINGS index_granularity = 8192,
         storage_policy = 'hot_cold';

-- Distributed 表（查询入口）
CREATE TABLE events_dist ON CLUSTER cluster AS events_local
ENGINE = Distributed(cluster, default, events_local, cityHash64(user_id));

-- 预聚合表（GMV/UV 按天+品类）+ 物化视图
CREATE TABLE events_daily_agg ON CLUSTER cluster (
    date        Date,
    category    LowCardinality(String),
    channel     LowCardinality(String),
    gmv_state   AggregateFunction(sum, Decimal(10,2)),
    uv_state    AggregateFunction(uniq, UInt64),
    pv          SimpleAggregateFunction(sum, UInt64)
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/events_daily_agg', '{replica}')
PARTITION BY toYYYYMM(date)
ORDER BY (date, category, channel);

CREATE MATERIALIZED VIEW mv_events_daily TO events_daily_agg AS
SELECT date, category, channel,
       sumState(amount) AS gmv_state,
       uniqState(user_id) AS uv_state,
       sum(toUInt64(1)) AS pv
FROM events_local GROUP BY date, category, channel;
```

### ⑤ 各场景查询路径

| 场景 | 查询路径 | 关键优化 |
|---|---|---|
| 实时大屏 GMV | `SELECT sumMerge(gmv_state) FROM events_daily_agg WHERE date=today()` | 预聚合，毫秒级 |
| BI 多维分析 | 查 `events_daily_agg`，GROUP BY category/channel | 预聚合 + 列存剪枝 |
| 用户行为轨迹 | `SELECT * FROM events_dist WHERE user_id=X ORDER BY dt` | 排序键 user_id 在前，单分片命中 + 二分剪枝 |
| 实时（秒级）大屏 | 直接查 `events_local` 最近几分钟（或加 Buffer 表） | 近实时 |

### ⑥ 高可用与容灾

```
• 写入可用性：Kafka 削峰 + 消费者重试 + 死信队列；CH 单分片宕机，Kafka 消息不丢
• 存储可用性：每分片 2 副本（ReplicatedMergeTree）+ Keeper 协调
• 跨机房：分片副本跨机房部署（或异步备份到异地 S3）
• 备份：每日 BACKUP 到 S3（复制不能防误删！）
• 降级：大促时关闭非核心查询、限制 max_concurrent_queries
```

### ⑦ 监控告警（必答）

```
• 写入：Kafka 积压、CH "Too many parts"、part 数、mutation 队列
• 复制：system.replicas.absolute_delay（副本延迟）
• 查询：system.query_log 慢查询、P95 延迟、OOM
• 资源：CPU/内存/磁盘/IO、Keeper 状态
• 业务：数据延迟（最新 event_time vs now）、数据完整性（行数对账）
```

### ⑧ 🔍 面试官追问与陷阱

- **追问 1：1000 万 QPS 真的能扛住吗？瓶颈在哪？**
  答：单靠 CH 扛不住原始 1000 万 QPS 写入。靠 **Kafka 削峰 + 攒批**：Kafka 抗住峰值，消费者攒批后每秒几百次 INSERT（每次几万行）落到 CH。瓶颈通常是 ①Kafka 分区数和消费者并行度；②CH 的 merge 速度（part 不能堆积）。

- **追问 2：为什么用 cityHash64(user_id) 分片，不用 rand()？**
  答：user_id 分片让**同一用户的事件落同一分片**，用户行为轨迹查询只命中单分片（不跨分片扫描），P95 低。rand() 虽均匀但无局部性，任何按 user_id 查询都要扫全部分片。

- **追问 3：预聚合用 uniqState，跨分片 UV 怎么算？会重复吗？**
  答：`uniqState` 底层是 HyperLogLog，**状态可合并**（union 操作）。各分片算本分片的 HLL 状态，initiator 用 `uniqMerge` 合并 → 跨分片去重正确。绝不能直接 `count(distinct)`（会把所有 uid 拉到 initiator 再算，内存爆炸 + 错误）。

- **追问 4：实时大屏要秒级，预聚合有延迟怎么办？**
  答：预聚合 MV 是插入时增量触发，延迟 = 攒批周期（秒级）。秒级大屏可接受。若要更快：①查明细表的最近 N 分钟（配合 Buffer 表攒批小写）；②近实时窗口用 ReplacingMergeTree + 频繁小批写入。

- **追问 5：90 天后删数据用什么方式？为什么不用 DELETE？**
  答：用 `TTL dt + 90 DAY DELETE` 或 `DROP PARTITION`。DROP PARTITION 是元数据操作（删整个月目录），极快。DELETE 是重量级 Mutation（重写 part），大表跑几天。**绝不用 DELETE 删大批量数据**。

- **追问 6：单分片数据倾斜（大用户）怎么办？**
  答：①监控各分片数据量（`system.parts` 按分片统计）；②大用户白名单单独路由到专属分片；③大用户内部再按 `item_id` 二次分片；④热点分离（热表 vs 冷表）。

- **追问 7：成本怎么控制？**
  答：①冷热分层（热 NVMe + 冷 S3，成本降 5~10x）；②预聚合减少明细存储；③压缩（ZSTD 冷数据）；④合理分片数（不是越多越好）；⑤查询缓存（重复报表）。

- **追问 8：数据丢了怎么办？重复了怎么办？**
  答：丢：Kafka 消费 offset 管理 + 死信队列 + CH 备份。重复：CH 对相同数据块有**去重机制**（INSERT 幂等，基于 data block hash），但只对相同批次的块去重；业务层用 event_id 去重（ReplacingMergeTree(event_id)）。

---

## 🎯 大题二：设计一个万亿级日志分析平台（替代 ELK）

### 需求
全公司应用/系统日志，日均 **1 万亿条**（~1PB/天压缩前），需支持：
1. **全文检索**：按关键词搜日志（类似 Kibana Discover）。
2. **聚合分析**：按 service/level/host 维度统计错误率、QPS。
3. **保留**：热 7 天，冷 30 天，归档 1 年。

### 架构关键决策

```
采集: Filebeat/Fluentd → Kafka（按 service 分区）
摄入: vector 攒批 → ClickHouse（按 service 分片）
存储: MergeTree + 【text 倒排索引】（v26.2 GA，替代 tokenbf）
      + 跳过索引 bloom_filter(service_id)
查询: Grafana（聚合）+ 自研检索 UI（全文）

表设计核心:
  ORDER BY (service_id, dt)        -- 排序键：服务在前（按服务查），时间在后
  INDEX idx_msg message TYPE text  -- 倒排索引加速全文检索
  INDEX idx_host host_id TYPE set(100) -- 跳过索引

容量估算:
  1万亿 × 200B ≈ 200TB/天（压缩前）
  列存+ZSTD 压缩 10x → 20TB/天
  7天热 ≈ 140TB（需 ~10 个分片，每分片 2TB NVMe）
  30天 → TTL MOVE 到 S3
```

### 🔍 追问陷阱

- **追问 1：CH 能替代 Elasticsearch 做全文检索吗？**
  答：能但定位不同。CH 的 `text` 倒排索引支持全文检索，但 **CH 不是搜索引擎**——没有 ES 的相关性评分、复杂 query DSL。CH 优势是**海量日志的聚合分析 + 低成本存储**（列存压缩远胜 ES）。纯检索高频用 ES，分析 + 检索混合用 CH，或 CH + ES 分工。

- **追问 2：日志写入非常稀疏（错误日志少），怎么避免 part 太多？**
  答：①Kafka 攒批（按时间或大小）；②不同 level 分表（error 表小且高频查，info 表大）；③Buffer 表攒批小写。

- **追问 3：全文检索慢怎么办？**
  答：①用 `text` 倒排索引（比 tokenbf 准且快）；②缩小时间范围（分区剪枝）；③按 service_id 先过滤（排序键）；④高频查询建 Projection。

---

## 🎯 大题三：设计一个实时用户画像/标签系统

### 需求
- **写入**：用户行为实时更新画像（亿级用户）。
- **查询**：①按 user_id **点查**画像（推荐系统实时用，P99 < 50ms）；②标签**聚合分析**（运营圈人，"高价值且活跃用户数"）。

### 架构关键决策（核心矛盾：点查 vs 聚合，CH 都要兼顾）

```
核心难点: CH 不擅长高 QPS 点查！点查交给 Redis，CH 做离线计算 + 聚合

架构:
  行为流 → CH（明细 + 计算标签）→ 定时/实时同步到 Redis（点查）
                 ↓
           画像宽表（聚合分析用）

表设计:
  -- 画像宽表（聚合分析）
  CREATE TABLE user_profile (
      user_id UInt64,
      reg_date Date,
      total_amount SimpleAggregateFunction(sum, Decimal),
      last_active DateTime,
      level LowCardinality(String),     -- 用户分级
      is_vip UInt8,
      tags Array(String)                -- 标签数组
  ) ENGINE = ReplacingMergeTree(version) -- 按 user_id 去重，留最新
  ORDER BY (user_id);

  -- 聚合圈人: SELECT count() FROM user_profile WHERE has(tags,'高价值') AND level='A'

  -- 点查: 不查 CH！查 Redis（CH 异步同步画像到 Redis）
```

### 🔍 追问陷阱

- **追问 1：为什么点查不用 CH？**
  答：CH 稀疏索引最细到 granule（8192 行），单点查也要扫 8192 行，单核 QPS 低（几百~几千）。推荐系统实时点查要 10 万+ QPS，CH 扛不住。用 Redis（纯内存哈希，百万 QPS）。

- **追问 2：CH 和 Redis 怎么同步？**
  答：①CH 算完画像后，用外部任务（如定时 SQL + 写 Redis）批量同步；②用 CH 的 MaterializedPostgreSQL 或 CDC；③实时性要求高时，行为流双写（CH + Redis 都写，或 CH + Kafka → Redis 消费者）。

- **追问 3：标签用 Array 还是每个标签一列？**
  答：①标签数少且固定 → 一列一标签（查询快、可索引）；②标签多且动态 → Array（灵活但查询用 `has()` 较慢，可加 bloom 跳过索引）；③超多标签 → 稀疏列 + 字典。

- **追问 4：圈人查询（count where tags）慢怎么办？**
  答：①标签列建 bloom_filter 跳过索引；②常用组合标签预计算成物化表；③`hasAll`/`hasAny` 函数优化；④冷热分层 + 合理分区。

---

## 系统设计通用答题框架（面试时套用）

```
1. 【澄清需求】写入量级？查询 SLA？数据保留？一致性要求？→ 显式列出假设
2. 【容量估算】行数 × 单行大小 × 压缩比 × 保留天数 → 存储量；QPS → 分片数
3. 【架构图】数据源 → 接入 → 缓冲(Kafka) → 摄入(攒批) → CH(分片+副本) → 查询
4. 【关键决策】分片键？排序键？分区？引擎？预聚合？冷热分层？
5. 【表设计】给出核心 DDL，解释每个选择的原因
6. 【高可用】副本、Keeper、备份、降级
7. 【监控】part 数、副本延迟、慢查询、资源
8. 【追问应对】预想陷阱：OOM、副本延迟、数据倾斜、丢数据、成本
```

**金句（面试加分）**：
- "写入吞吐靠**分片水平扩展**，高可用靠**副本**，加速查询靠**排序键 + 预聚合**，省成本靠**冷热分层 + 压缩**。"
- "ClickHouse 是 OLAP 不是 OLTP，**点查和高频小写是它的反模式**，要靠 Kafka 攒批 + Redis 缓存补足。"
- "**复制不能代替备份**，误操作会同步到所有副本。"

---

# 📚 资料来源（权威参考）

**官方文档**
- MergeTree 表引擎：https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/mergetree
- 稀疏主索引实战指南：https://clickhouse.com/docs/guides/clickhouse/data-modelling/sparse-primary-indexes
- 架构学术总览（向量化/三层并行/复制模型）：https://clickhouse.com/docs/concepts/core-concepts/academic-overview
- ReplacingMergeTree / CollapsingMergeTree：https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/replacingmergetree
- JOIN 语法与严格度：https://clickhouse.com/docs/en/sql-reference/statements/select/join
- 向量化查询执行：https://clickhouse.com/resources/engineering/vectorized-query-execution
- Projection：https://clickhouse.com/docs/en/sql-reference/statements/alter/projection
- Tiered Storage / Storage Policies：https://clickhouse.com/docs/en/operations/storing-data
- BACKUP/RESTORE：https://clickhouse.com/docs/en/operations/backup

**大厂 / 社区博客**
- Altinity KB - Collapsing vs Replacing：https://kb.altinity.com/engines/mergetree-table-engine-family/collapsing-vs-replacing/
- Altinity - MergeTree 内核讲义 PDF：https://altinity.com/wp-content/uploads/2023/11/ClickHouse-Data-Management-Internals-MergeTree-Storage-Merges-Replication-2023-11-15.pdf
- Altinity - ClickHouse on S3：https://altinity.com/blog/clickhouse-mergetree-on-s3-intro-and-architecture
- Tinybird - ReplacingMergeTree best use cases：https://www.tinybird.co/blog/clickhouse-replacingmergetree-example
- OneUptime - Sparse Primary Index：https://oneuptime.com/blog/post/2026-03-31-clickhouse-sparse-primary-index/view
- bigdataboutique - MergeTree Engine：https://bigdataboutique.com/blog/clickhouse-mergetree-engine
- manumartinm - Minimal MergeTree from Scratch：https://manumartinm.dev/blog/clickhouse-mergetree

---

> **使用建议**：面试时先答"标准答案"展示知识框架，被追问时深入"🔍 追问"部分体现深度。每个"踩分点"是面试官心里的加分项，务必主动提到。陷阱题快查表适合面试前一晚突击。
