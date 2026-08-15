# MergeTree 引擎选型实战指南 —— "我的业务该用哪个？为什么慢？"

> 这篇解决三个实际问题：
> ① 六个 MergeTree 变体到底怎么选（按**数据形态**选，不是背参数）
> ② "为什么我用这个引擎查询很慢"（每个引擎的性能因果链）
> ③ 领导/开发质问时的标准回答话术

---

## 第 1 章：一个心智模型看懂全家桶（先懂这个，后面全通）

**所有 MergeTree 变体 = 普通 MergeTree + "后台合并时顺手多做一件事"**

```
                    MergeTree（普通版）
                 数据按批写入 → part → 后台合并
                            │
        ┌──────────┬────────┼─────────┬──────────┬──────────┐
        ▼          ▼        ▼         ▼          ▼          ▼
   合并时     合并时     合并时     合并时      合并时     什么都不
   去重留最新  +1/-1对冲  带版本对冲  数值列SUM   合并聚合    多做
        │          │        │         │          │          │
 Replacing  Collapsing Versioned  Summing   Aggregating   普通
 MergeTree  MergeTree Collapsing  MergeTree MergeTree    MergeTree
                     MergeTree
```

**三条铁律（所有"慢"和"脏数据"的根源都在这）：**

```
铁律①：特殊逻辑发生在【后台合并时】，不在写入时、也不在查询时
        → 异步、最终生效、没有时间保证

铁律②：去重/折叠只对【同一个 part 内】的数据生效
        → 还没合并到一起的重复行，查询时照样看得见

铁律③：合并要干额外的事 → 写放大（merge 更忙、更耗 CPU/IO）
        → 特殊逻辑越复杂，写入吞吐越低
```

记住这三条，后面每个"为什么慢"你都能自己推出来。

---

## 第 2 章：六个引擎逐个讲透（每个都按同一套模板）

> 模板：它解决什么问题 → 生活类比 → 什么时候用 → DDL 长什么样 → 查询必须怎么写 → 什么情况下会慢

### 2.1 MergeTree（普通版）—— 默认选择

**解决什么**：什么都不解决，就是快。纯追加、不更新。
**类比**：只进货从不错货的仓库。
**什么时候用**：日志、埋点、事件流、行为记录——**写入后永不修改**的数据。生产 70% 的场景就是它。

```sql
CREATE TABLE events (
    event_time DateTime,
    user_id    UInt64,
    event_type LowCardinality(String),
    payload    String
) ENGINE = MergeTree
PARTITION BY toYYYYMM(event_time)
ORDER BY (user_id, event_time);
```

**查询**：正常写，无任何限制。
**什么时候慢**：跟引擎无关，慢通常是排序键没设计好（WHERE 没命中 ORDER BY 前列）。

⚠️ **最大误区**：很多人无脑用 ReplacingMergeTree"以防数据重复"——错！不需要更新就别用变体，白白承担合并开销和查询限制。

---

### 2.2 ReplacingMergeTree —— "同一主键留最新版本"

**解决什么**：**upsert 语义**——同一主键写多次，最终只留最新一条。
**类比**：花名册，同名的人只留最新登记的那份资料。
**什么时候用**：数据会**按主键重复写入且以最新为准**——订单状态、用户资料、商品价格、设备最新状态。

```sql
CREATE TABLE orders (
    order_id    UInt64,
    status      LowCardinality(String),   -- 待支付→已支付→已发货
    amount      Decimal(10,2),
    version     UInt64,                    -- ★版本列：时间戳或自增ID
    updated_at  DateTime
) ENGINE = ReplacingMergeTree(version)     -- ★指定版本列，留 version 最大的
ORDER BY (order_id);                       -- ★ORDER BY 就是"主键"=去重键
```

**查询必须怎么写（这是 90% 翻车点）**：
```sql
-- ❌ 错误认知：以为查出来自动去重 → 会出现重复行+旧状态！
SELECT * FROM orders WHERE order_id = 1001;

-- ✅ 方案A：argMax（推荐，性能好）
SELECT order_id,
       argMax(status, version) AS status,   -- 取 version 最大那行的 status
       argMax(amount, version) AS amount
FROM orders
GROUP BY order_id;

-- ✅ 方案B：FINAL（简单但慢，只适合低频/小表）
SELECT * FROM orders FINAL WHERE order_id = 1001;
```

**为什么慢（因果链）**：
```
FINAL 的代价：强制在查询时"虚拟合并"所有 part → 无法充分并行 → part 越多越慢
  100 个 part 的表上 FINAL 可能比 argMax 慢 10~50 倍
频繁更新同一主键的代价：每次更新=多一行版本 → 数据膨胀 → merge 频繁重写
  订单状态流转 10 次 = 10 行数据，merge 永远在忙
```

**领导质问"怎么查出来两条订单"的标准答案**：
> "这是 ReplacingMergeTree 的设计行为：去重发生在后台合并，是异步最终生效的。我们查询层统一用 argMax 保证结果正确，用 FINAL 会牺牲性能。"

---

### 2.3 SummingMergeTree —— "合并时数值列自动求和"

**解决什么**：**预聚合数字**。按 ORDER BY 键合并时，数值列自动 SUM。
**类比**：零钱罐，同面额的硬币自动合并成总额。
**什么时候用**：查询只关心**加法汇总**的场景——金额累计、计数、按维度的指标累计。写入端往往已经按维度聚合过一轮（如分钟级指标）。

```sql
CREATE TABLE metrics_minute (
    metric_name  LowCardinality(String),
    minute       DateTime,
    value        Float64,        -- ★数值列会被自动 SUM
    cnt          UInt64
) ENGINE = SummingMergeTree
ORDER BY (metric_name, minute);
```

**查询必须怎么写**：
```sql
-- ⚠ 即使合并了，不同 part 可能还有同键行 → 查询必须再 SUM 一次（幂等）
SELECT metric_name, sum(value)   -- ★必须 sum，不能直接 SELECT value
FROM metrics_minute
WHERE minute >= now() - INTERVAL 1 HOUR
GROUP BY metric_name;
```

**为什么慢 / 出错**：
```
① 直接 SELECT value 不 GROUP BY → 拿到的是"某个 part 的部分和"，数据错！
② 非数值列不会被聚合 → 同键不同值的字符串列取【任意行】→ 结果不确定！
   （字符串维度列要么放进 ORDER BY，要么保证同键同值）
```

**什么时候不用它**：需要 `uniq`（去重计数）、`quantile`（分位数）时——SUM 无法合并这些聚合，要用 AggregatingMergeTree。

---

### 2.4 AggregatingMergeTree —— "合并时聚合状态自动合并"（预聚合之王）

**解决什么**：**通用预聚合**。存"聚合中间状态"（不只是 SUM，还有去重计数、分位数等），合并时状态自动合并。
**类比**：不做完整统计，先把"半成品统计"存起来，随合并不断融合，查询时最后收口。
**什么时候用**：**明细数据量巨大，但查询只要聚合结果**——实时大屏（GMV/UV）、多维报表、用户行为漏斗。**标准搭配：物化视图 + State/Merge 函数**。

```sql
-- ① 明细表（普通 MergeTree，亿级行/天）
CREATE TABLE events (...省略...) ENGINE = MergeTree ORDER BY (user_id, event_time);

-- ② 预聚合目标表（AggregatingMergeTree）
CREATE TABLE events_agg (
    day        Date,
    channel    LowCardinality(String),
    uv_state   AggregateFunction(uniq, UInt64),   -- ★存"状态"不是存值！
    gmv_state  AggregateFunction(sum, Decimal(10,2))
) ENGINE = AggregatingMergeTree
ORDER BY (day, channel);

-- ③ 物化视图：明细写入时自动增量聚合到目标表
CREATE MATERIALIZED VIEW mv_events_agg TO events_agg AS
SELECT toDate(event_time) AS day,
       channel,
       uniqState(user_id)  AS uv_state,    -- ★State：生成聚合状态
       sumState(amount)    AS gmv_state
FROM events GROUP BY day, channel;
```

**查询必须怎么写**：
```sql
SELECT day, channel,
       uniqMerge(uv_state)  AS uv,     -- ★Merge：把状态合并成最终结果
       sumMerge(gmv_state)  AS gmv
FROM events_agg GROUP BY day, channel; -- 亿级明细 → 毫秒级查询
```

**为什么慢 / 出错**：
```
① 直接 SELECT * 看到的是二进制状态 → "怎么是乱码？"（不是乱码，是状态）
② 忘了用 uniqMerge 用了 uniq(uv_state) → 类型报错或错误结果
③ 明细表查询压力没减 → 有人直接查明细表 → 白建了预聚合
④ State 状态占空间比普通数值大（uniq 的 HLL 状态约几 KB/组）
   → 分组基数极高时（如按 user_id 分组），预聚合反而费空间 → 别滥用
```

---

### 2.5 CollapsingMergeTree —— "用 +1/-1 行对冲模拟更新"

**解决什么**：把**更新/删除**变成**插入两条记录**（sign=-1 抵消旧行 + sign=+1 新行），合并时正负抵消。
**类比**：会计记账——改错账不撕账本，写一笔红字冲销再记正确的。
**什么时候用**：需要**精确的"状态当前值"和"历史变更"都要保留**的场景——购物车、会话状态、账户余额流水。

```sql
CREATE TABLE user_sessions (
    session_id UInt64,
    state      LowCardinality(String),
    sign       Int8                  -- ★+1=有效行，-1=取消行
) ENGINE = CollapsingMergeTree
ORDER BY (session_id);
```

**为什么慢 / 出错（坑最多，能用 Versioned 替代就别用它）**：
```
① 写入要求【同会话有序】：-1 抵消行必须和旧行在【同一批次/同一 partition block】写入
   → 多线程/多客户端乱序写 → 抵消失败 → 数据膨胀+查错
② 查询必须带 sum(sign)：
   SELECT session_id, sum(state * sign) ... 或 WHERE 后聚合过滤
   → 所有 SQL 都要改写
③ 抵消失败排查极难 → 生产事故高发区
```

---

### 2.6 VersionedCollapsingMergeTree —— Collapsing 的工业级修正版

**解决什么**：和 Collapsing 一样，但多了 **version 列**，允许乱序写入——抵消不靠"批次顺序"，靠"版本号配对"。
**类比**：红字冲销不再要求"当面对质"，凭单据号（版本）就能配对。
**什么时候用**：所有想用 Collapsing 的场景，**生产直接用它**（多机/多线程写入不会翻车）。

```sql
ENGINE = VersionedCollapsingMergeTree(version)   -- version 是 UInt64 版本列
ORDER BY (session_id);
-- 写入：改状态 = 插入 (旧行, sign=-1, version=N) + (新行, sign=+1, version=N)
--       同一 version 的一对正负行合并时抵消
```

**为什么慢**：同 Collapsing（合并逻辑更复杂一点），且查询仍需 sum(sign)。

---

## 第 3 章：选型决策树（直接背这个）

```
问自己第一个问题：【写入后，数据需要"修改/去重"吗？】

├── 不需要（日志/埋点/事件，写完就不变）
│     └── ✅ MergeTree（默认。别多想，70% 场景就是它）
│
├── 需要"同一主键多次写，以最新为准"（upsert）
│     ├── 查询层能接受改 SQL（argMax）
│     │     └── ✅ ReplacingMergeTree(version)
│     │         （订单状态、用户资料、商品价格、设备最新值）
│     └── 需要保留变更历史 + 精确当前值
│           └── ✅ VersionedCollapsingMergeTree
│
├── 需要"状态机式流转"且高并发写入（购物车/会话/余额）
│     └── ✅ VersionedCollapsingMergeTree（别用裸 Collapsing）
│
└── 明细巨大，查询只要聚合结果（大屏/报表/漏斗）
      ├── 只要 SUM/计数
      │     └── ✅ SummingMergeTree（简单场景够用）
      └── 要 uniq / quantile / 任意聚合
            └── ✅ AggregatingMergeTree + 物化视图（State/Merge）
```

**三步选型口诀**：
1. **不更新 → 普通 MergeTree**
2. **要最新 → Replacing（配 argMax）**；要历史+精确 → VersionedCollapsing
3. **要汇总 → SUM 用 Summing，复杂聚合用 Aggregating+MV**

---

## 第 4 章：业务场景对照表（拿着对号入座）

| 业务场景 | 数据形态 | 选型 | 查询写法要点 | 主要坑 |
|---|---|---|---|---|
| 埋点/日志/审计 | 只追加永不改 | **MergeTree** | 正常查 | 别多此一举用变体 |
| 订单状态跟踪 | 同订单多版本留最新 | **ReplacingMergeTree(ver)** | argMax 或 FINAL | 直接 SELECT 出重复行 |
| 用户画像/资料 | 同用户多版本留最新 | **ReplacingMergeTree(ver)** | argMax | FINAL 大表慢几十倍 |
| 商品价格快照 | 同商品多价格留当前 | **ReplacingMergeTree(ver)** | argMax | 同上 |
| 实时大屏 GMV/UV | 明细巨大只要聚合 | **AggregatingMergeTree + MV** | uniqMerge/sumMerge | 忘写 Merge 函数 |
| 分钟级监控指标 | 已按分钟预聚合数值 | **SummingMergeTree** | 查询再 sum | 字符串列取值不确定 |
| 购物车/会话状态 | 状态频繁流转+要历史 | **VersionedCollapsing** | sum(sign) | 写入端配对逻辑复杂 |
| CDC 同步 MySQL 表 | upsert+delete 语义 | **ReplacingMergeTree(ver)** / Collapsing 系 | argMax | 删除语义要专门处理 |

---

## 第 5 章："为什么我用这个很慢？"对照表（领导质问速查）

| 现象 | 引擎 | 根因 | 解法 |
|---|---|---|---|
| 查询越跑越慢，part 多时尤其明显 | Replacing + FINAL | FINAL 强制查询时虚拟合并，part 越多越慢 | 换 argMax；控制 part 数 |
| 明明该 1 条，查出 N 条 | Replacing | 去重是异步的，重复行还在不同 part | 查询层 argMax 兜底 |
| 写入吞吐莫名比普通表低 | 任何变体 | 合并时干额外的事 = 写放大 | 评估是否真的需要该变体 |
| 数据越写越膨胀 | Collapsing | 乱序写入抵消失败，正负行都留着 | 换 VersionedCollapsing |
| 字符串列查出"随机值" | Summing | 非数值列合并时取任意行 | 维度列全进 ORDER BY |
| 查出"乱码"/类型报错 | Aggregating | 直接 SELECT 了聚合状态列 | 必须 uniqMerge/sumMerge |
| 大屏查询还是很慢 | 建了 Aggregating 但没人用 | 查询还在打明细表 | 查询切到聚合表 |
| uv 状态占空间巨大 | Aggregating | 分组基数太高（如按 user_id） | 分组维度别用高基数列 |

**排查心法**：先确认"**引擎特性和查询方式是否匹配**"（80% 的慢/错都是这个），再看排序键/分区设计，最后看资源。

---

## 第 6 章：汇报话术（向领导/开发解释时用）

**话术 1：为什么选 Replacing 而不是普通表？**
> "我们的数据是 upsert 语义（同一订单会多次更新状态），普通 MergeTree 不去重会查出重复。ReplacingMergeTree 在后台合并时自动按主键留最新版本，写入吞吐不受影响，代价只是查询层要用 argMax 取最新值。"

**话术 2：为什么不用 FINAL？**
> "FINAL 会在查询时强制做虚拟合并，100 个 part 的表上比 argMax 慢 10 到 50 倍，而且并发一高就把 CPU 打满。我们统一用 argMax + version，性能可控。FINAL 只留给低频对账场景。"

**话术 3：为什么大屏要建物化视图 + AggregatingMergeTree？**
> "明细表一天 50 亿行，直接聚合要扫全量。物化视图在写入时增量预聚合，查询打聚合表，从秒级降到毫秒级。这是 ClickHouse 官方推荐的标准预分层架构。"

**话术 4：为什么这个表越查越慢？**
> "两个可能：一是数据版本堆积，重复行还没合并，查询做了额外过滤；二是 part 数量上涨导致合并压力变大。我先看 system.parts 和 query_log 的 read_rows 确认是哪种，再对应处理。"（体现排查方法论，不是甩锅）

---

## 第 7 章：一张图总结（打印贴工位）

```
┌─────────────────────────────────────────────────────────────┐
│                数据需要更新/去重/预聚合吗？                     │
└──────┬──────────────┬──────────────┬────────────────────────┘
   不需要          留最新版本       要聚合结果
       │              │              │
       ▼              ▼              ▼
   MergeTree    ReplacingMT      只要SUM→SummingMT
   (默认70%)    +argMax查询       复杂聚合→AggregatingMT+MV
                     │
              要历史+精确/高并发状态流转
                     │
                     ▼
             VersionedCollapsingMT
            （别用裸 CollapsingMT）

三条铁律：①特殊逻辑在合并时异步生效 ②只对同part生效
         ③合并多干活=写放大
一句话：查询方式必须配合引擎特性，用错查询方式=慢+错
```
