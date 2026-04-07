# Fluss 存储设计：从 Kafka 日志结构到 Fluss Log+KV 一体化

---

## 前言：Kafka 日志设计原理

在介绍 Fluss 之前，我们先来理解 Kafka 的日志设计原理，因为 Fluss 的 LogStore 设计很大程度上借鉴了 Kafka 的优秀设计。

### 1.1 Kafka 整体日志结构

Kafka 是一个分布式消息队列，其存储核心就是**日志结构**：

```
Topic (主题)
  └─ Partition (分区)
      └─ Log (日志)
          ├─ Segment 0  (baseOffset = 0)
          │   ├─ 0000000000000000000.log      (数据文件)
          │   ├─ 0000000000000000000.index    (偏移量索引)
          │   └─ 0000000000000000000.timeindex (时间戳索引)
          ├─ Segment 1  (baseOffset = 13584)
          │   ├─ 0000000000000013584.log
          │   ├─ 0000000000000013584.index
          │   └─ 0000000000000013584.timeindex
          └─ ...
```

### 1.2 核心设计原理

#### （1）分段设计

Kafka 将一个 Partition 的日志切分成多个**固定大小**的 Segment（默认 1GB）。

**为什么要分段？**

- **方便删除**：旧数据到期后，可以直接整个 Segment 文件删除，不需要修改现有文件
- **查找高效**：可以快速二分查找定位到哪个 Segment 包含目标 offset
- **并发更好**：读和写不冲突，写入只在 active segment，读取可以在其他 segment 并发进行

**默认参数（以 Kafka 2.8 为例）：**

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `log.segment.bytes` | 1GB | 单个日志分段最大大小 |
| `log.segment.ms` | 7天 | 分段最长保留时间 |
| `log.index.interval.bytes` | 4096 (4KB) | 每隔多少字节写入一个索引条目 |
| `log.index.size.max.bytes` | 10MB | 索引文件最大大小 |

#### （2）稀疏索引

Kafka 不为每条记录建立索引，而是**每隔 4KB 数据建立一个索引条目**，这就是稀疏索引。

**索引文件结构（offset index）：**

| 相对offset (4字节) | 物理位置 (4字节) |
|-------------------|----------------|
| 0 | 0 |
| 346 | 15234 |
| 879 | 32456 |
| ... | ... |

**为什么用稀疏索引？**

- **节省空间**：假设 1GB 日志，每 4KB 一个索引，只需要 `1GB / 4KB = 262,144` 个索引条目，每个条目 8 字节，总共只需要 **2MB**
- **查找依然高效**：先通过二分查找找到小于目标 offset 的最大索引条目，然后从那个位置开始顺序扫描到目标 offset，因为间隔只有 4KB，顺序扫描很快

**举个具体例子：**

假设我们要查找 offset = 1000 的消息：

1. 二分查找索引文件，找到最大的不大于 1000 的索引条目，比如相对 offset = 879 在物理位置 32456
2. 从物理位置 32456 开始顺序扫描，直到找到 offset = 1000
3. 总共只需要扫描 1000 - 879 = 121 条消息，非常快

#### （3）顺序写入

Kafka 只支持**顺序追加写入**，不支持修改已写入的数据。

```
写入新消息 → 追加到 active segment 的末尾 → 完成
```

**为什么顺序写入这么快？**

- 磁盘顺序写入比随机写入快 **100x** 以上（机械硬盘：顺序写入 ~200MB/s，随机写入 ~2MB/s）
- 充分利用操作系统 page cache，批量写入聚合 IO
- 没有磁盘碎片问题

**具体数据对比（7200 RPM SATA 机械硬盘）：**

| 方式 | 吞吐量 |
|------|--------|
| 随机写 | ~2 MB/s |
| 顺序写 | ~200 MB/s |
| **差距** | **100x** |

#### （4）页缓存（Page Cache）

Kafka 充分利用操作系统页缓存：

- 读取时：最新数据在 page cache，直接返回，不需要读磁盘
- 写入时：先写入 page cache，后台异步刷盘
- 操作系统会自动管理哪些内容留在内存，哪些换出到磁盘

**性能提升**：对于热数据，读取命中率可以达到 99% 以上，几乎都是内存读取，速度极快。

#### （5）零拷贝（Zero-Copy）

Kafka 利用 `sendfile` 系统调用实现零拷贝传输：

```
磁盘数据 → page cache → socket缓冲区 → 网卡
                ↓
跳过了 → 复制到用户空间缓冲区这一步
```

**节省了什么？**
- 避免了内核空间 ↔ 用户空间的数据拷贝
- 减少了 CPU 上下文切换
- 大数据量传输时性能提升显著

### 1.3 Kafka 查找流程举例

假设我们要消费从 offset = 15000 开始的消息：

```
1. 根据 offset 15000，二分查找找到哪个 Segment 包含它
   → Segment 的 baseOffset 一定 ≤ 15000，下一个 Segment baseOffset > 15000

2. 在找到的 Segment 中，计算相对 offset = 15000 - baseOffset

3. 二分查找偏移量索引文件，找到最大的不大于相对 offset 的索引条目
   → 得到这个索引条目对应的物理位置

4. 从这个物理位置开始，顺序扫描 log 文件，直到找到目标 offset

5. 开始读取数据返回给客户端
```

整个过程只需要 **2 次二分查找** + 少量顺序扫描，非常高效。

---

## Fluss 表设计概述

Fluss 提供两种核心表类型，并通过不同的合并引擎满足不同业务场景需求：

```
Fluss Table
├── 日志表 (Log Table) - 纯流式写入，仅追加
└── 主键表 (Primary Key Table) - 支持更新删除，Log + KV 一体化
    ├── Merge Engine 决定相同主键如何合并
    │   ├── 默认合并 (Default) - 最后一条胜出
    │   ├── 首行保留 (First Row) - 第一条胜出
    │   ├── 版本化 (Versioned) - 保留多个版本
    │   └── 聚合 (Aggregation) - 增量聚合
```

## 一、日志表（Log Table）

### 1.1 设计理念

**日志表**是 Fluss 最基础的表类型，设计目标是**高吞吐流式写入**，类似 Kafka。

特点：
- 仅支持**追加写入**，不支持更新和删除
- 数据持久化存储在 LogStore
- 不依赖 KvStore，存储开销更小
- 支持无限扩容，百万级 TPS 写入

### 1.2 适用场景

| 场景 | 说明 |
|------|------|
| **事件流存储** | 用户行为日志、IoT 设备数据、点击流 |
| **CDC 日志管道** | 数据库 Binlog 同步后的存储，供下游消费 |
| **消息队列替代** | 作为 Kafka 替代，存储业务事件 |
| **数据湖入口** | 原始数据落地，再异步合并到数据湖 |

### 1.3 存储结构

```
Log Table
  ├─ Partition (可选)
  │   └─ Bucket N
  │       └─ LogTablet (on TabletServer)
  │           ├─ Segment 0
  │           │   ├─ {baseOffset}.log
  │           │   ├─ {baseOffset}.index
  │           │   └─ {baseOffset}.timeindex
  │           ├─ Segment 1
  │           └─ ...
  └─ ...
```

**与 Kafka 的对比**：

| 特性 | Kafka | Fluss LogTable |
|------|-------|----------------|
| 分段索引 | ✅ 类似 Kafka | ✅ 借鉴 Kafka，相同设计 |
| 稀疏索引 | ✅ | ✅ |
| 顺序写入 | ✅ | ✅ |
| 多格式支持 | ❌ 只有一种 | ✅ ARROW/INDEXED/COMPACTED |
| 分层存储 | 需要第三方工具（Tiered Storage） | ✅ 原生支持 |
| 湖仓集成 | ❌ 不原生支持 | ✅ 原生支持自动下沉 |

### 1.4 配置示例

```sql
-- 创建用户行为日志表
CREATE TABLE user_behavior (
    user_id BIGINT,
    event_time TIMESTAMP,
    event_type STRING,
    page_url STRING,
    referrer STRING,
    device STRING
) WITH (
    'log.format' = 'arrow',        -- 列式存储，适合分析
    'bucket.num' = '128',          -- 128 个分桶
    'replication.factor' = '2',    -- 2副本
    'data-lake.enabled' = 'true'  -- 开启下沉到数据湖
);
```

## 二、主键表（Primary Key Table）

### 2.1 设计理念

**主键表**是 Fluss 最具特色的设计，采用 **Log + KV 一体化架构**：

- **LogStore**：存储完整的变更日志（changelog），相当于 WAL，支持 CDC 消费，用于故障恢复
- **KvStore**：存储主键 → 最新值的快照，支持高效点查询和更新

```
写入流程：
写入一条更新 → 写入内存缓冲 → 写入 LogStore（WAL）→ 返回成功 → 异步刷入 KvStore

读取流程：
点查 → 先查内存缓冲 → 未命中 → 查 RocksDB → 返回结果
```

### 2.2 为什么要 Log + KV 分离？

| 好处 | 说明 |
|------|------|
| **完整变更历史** | LogStore 保存了所有变更，可以产出完整 CDC，供下游消费 |
| **故障恢复** | 如果 KvStore 损坏，可以从 LogStore 重放恢复 |
| **两种访问模式** | 既支持流式消费变更，又支持随机点查询 |
| **写入优化** | 写入先缓冲，批量刷入 RocksDB，减少随机写 |

### 2.3 存储结构

```
Primary Key Table
  ├─ Partition (可选)
  │   └─ Bucket N
  │       └─ Tablet (on TabletServer)
  │           ├─ LogTablet (LogStore)
  │           │   └─ 分段日志（同日志表）存储 changelog
  │           └─ KvTablet (KvStore，基于 RocksDB)
  │               └─ RocksDB SST 文件存储主键 → 值映射
  └─ ...
```

**关键设计点**：
- 同一个 Bucket 的 LogTablet 和 KvTablet 一定分配在同一个 TabletServer，避免跨网访问
- LogTablet 支持多副本，KvTablet 当前不支持副本，依赖快照从远程恢复
- KvTablet 支持增量快照，故障恢复只需要重放快照之后的日志

### 2.4 Changelog 两种模式

| 模式 | 存储内容 | 适用场景 | 空间占用 |
|------|----------|----------|----------|
| **FULL** | 更新保存 BEFORE + AFTER 两个镜像 | 需要完整 CDC，审计场景 | 较大 |
| **WAL** | 只保存 AFTER 镜像 | 仅需要恢复，不需要CDC | 较小 |

### 2.5 适用场景

| 场景 | 说明 |
|------|------|
| **实时维度表** | Flink 实时计算维表，需要点查 |
| **用户画像表** | 存储用户标签，频繁更新 |
| **订单主表** | 需要更新订单状态，支持按订单号点查 |
| **实时特征表** | 机器学习实时特征存储 |
| **维度建模事实表** | 拉链表，需要缓存在实时层 |

### 2.6 配置示例

```sql
-- 创建用户维度主键表
CREATE TABLE dim_user (
    user_id BIGINT,
    username STRING,
    nickname STRING,
    email STRING,
    phone STRING,
    gender INT,
    register_time TIMESTAMP,
    PRIMARY KEY (user_id) NOT ENFORCED
) WITH (
    'log.format' = 'arrow',         -- 列式存储 changelog
    'kv.format' = 'compacted',      -- 紧凑存储 KV
    'changelog-image' = 'full',     -- 保存完整变更镜像
    'bucket.num' = '256',
    'replication.factor' = '2'
);
```

## 三、合并引擎（Merge Engines）

对于主键表，当多条数据有**相同主键**时，需要合并引擎决定保留哪条数据。Fluss 提供四种合并引擎。

### 3.1 默认合并引擎（Default Merge Engine）

**原理**：**最后写入的数据胜出**（Last Write Wins）

这是最常用的合并策略，对于大部分更新场景，都希望最新的数据覆盖旧数据。

**适用场景**：
- 实时维度表
- 用户画像
- 订单状态更新
- 任何需要"最新版本"的场景

**行为举例**：

| 顺序 | 主键 | 数据 | 合并后结果 |
|------|------|------|------------|
| 1 | 1001 | 张三 | {1001: 张三} |
| 2 | 1001 | 张三疯 | {1001: 张三疯} |
| 3 | 1001 | 张三丰 | {1001: 张三丰} |

**SQL 配置**：
```sql
CREATE TABLE orders (
    order_id BIGINT,
    status STRING,
    amount DECIMAL(18,2),
    update_time TIMESTAMP,
    PRIMARY KEY (order_id) NOT ENFORCED
) WITH (
    'merge.engine' = 'default'  -- 默认可以不写，就是这个
);
```

### 3.2 首行合并引擎（First Row Merge Engine）

**原理**：**第一条写入的数据胜出**，后续相同主键的数据都被忽略。

**适用场景**：
- 去重表
- 事件首次触发性统计
- 数据入库时保留原始记录，忽略后续更新

**行为举例**：

| 顺序 | 主键 | 数据 | 合并后结果 |
|------|------|------|------------|
| 1 | 1001 | 首次注册信息 | {1001: 首次注册信息} |
| 2 | 1001 | 更新信息 | {1001: **首次注册信息**}（不变）|
| 3 | 1001 | 再次更新 | {1001: 首次注册信息}（仍然不变）|

**典型用例：用户首次事件统计**
```sql
-- 统计用户首次登陆时间，重复记录不改变结果
CREATE TABLE user_first_login (
    user_id BIGINT,
    first_login_time TIMESTAMP,
    PRIMARY KEY (user_id) NOT ENFORCED
) WITH (
    'merge.engine' = 'first-row'
);

-- 不管输入多少条相同 user_id，first_login_time 永远保留第一条
```

### 3.3 版本化合并引擎（Versioned Merge Engine）

**原理**：保留同一个主键的**多个版本**，可以查询历史版本。

版本由用户指定的版本列区分，版本列必须是单调递增的（比如 version_id, update_time）。

**适用场景**：
- 缓慢变化维
- 历史轨迹查询
- 审计需求
- 可回滚更新

**行为举例**：

假设版本列是 `version_id`：

| 顺序 | 主键 | version_id | 数据 | 存储结果 |
|------|------|------------|------|----------|
| 1 | 1001 | 1 | 地址A | 存 [(1, 地址A)] |
| 2 | 1001 | 2 | 地址B | 存 [(1, 地址A), (2, 地址B)] |
| 3 | 1001 | 3 | 地址C | 存 [(1, 地址A), (2, 地址B), (3, 地址C)] |

查询可以指定查哪个版本，或者查询最新版本。

**SQL 配置**：
```sql
-- 缓慢变化用户维度表，保留地址变更历史
CREATE TABLE dim_user_history (
    user_id BIGINT,
    address STRING,
    version_id INT,
    update_time TIMESTAMP,
    PRIMARY KEY (user_id) NOT ENFORCED
) WITH (
    'merge.engine' = 'versioned',
    'versioned.version-column' = 'version_id'  -- 指定版本列
);
```

### 3.4 聚合合并引擎（Aggregation Merge Engine）

**原理**：对相同主键进行**增量聚合**，支持累加、计数等。

每条新记录进来，不是直接覆盖，而是和已有值做聚合计算。

**适用场景**：
- 实时统计聚合
- 实时指标计算
- 增量累加计数

**支持的聚合类型**：
| 聚合类型 | 说明 |
|----------|------|
| `sum` | 累加数值 |
| `count` | 计数 |
| `max` | 保留最大值 |
| `min` | 保留最小值 |
| `last_non_null` | 保留最后一个非空值 |
| `first_non_null` | 保留第一个非空值 |

**行为举例**：
- **SUM 聚合**：

| 顺序 | 主键 | 金额 | 聚合结果 |
|------|------|------|----------|
| 1 | 1001 | 100 | 100 |
| 2 | 1001 | 50 | 150 |
| 3 | 1001 | 30 | 180 |

- **MAX 聚合**：

| 顺序 | 主键 | 点击量 | 聚合结果 |
|------|------|--------|----------|
| 1 | 1001 | 10 | 10 |
| 2 | 1001 | 15 | 15 |
| 3 | 1001 | 12 | 15 |

**SQL 配置**：
```sql
-- 实时统计每个用户日消费金额
CREATE TABLE daily_user_spending (
    dt DATE,
    user_id BIGINT,
    spending DECIMAL(18,2),
    order_count INT,
    PRIMARY KEY (dt, user_id) NOT ENFORCED
) WITH (
    'merge.engine' = 'aggregation',
    'aggregation.aggregations' = 'spending:sum,order_count:sum'
);
```

### 3.5 合并引擎对比

| 合并引擎 | 保留几条 | 适用场景 | 空间开销 |
|----------|---------|----------|----------|
| default | 1条（最后）| 大多数更新场景 | 最小 |
| first-row | 1条（第一条）| 去重、首次事件 | 最小 |
| versioned | N条（所有版本）| 历史追溯、审计 | 最大 |
| aggregation | 1条（聚合结果）| 实时聚合指标 | 小 |

## 四、三种日志格式对比

Fluss 支持三种日志格式，面向不同 workload 优化：

| 格式 | 存储方向 | 默认 | 适用场景 |
|------|----------|------|----------|
| **ARROW** | 列式存储 | ✅ 是 | 实时分析、Ad-hoc 查询、湖仓集成 |
| **INDEXED** | 行式存储 | 否 | 事件驱动、流式消费、小批量写入 |
| **COMPACTED** | 紧凑行式 | 否 | 主键表、空间受限场景、读少写多 |

### 4.1 ARROW（列式存储）

**特点**：
- 基于 Apache Arrow 列式格式
- 支持列裁剪（查询只读需要的列）
- 压缩效率高
- 分析型查询性能好

**适用示例**：
```sql
-- 实时数仓事实表，经常做分析查询
CREATE TABLE sales.fact_order (
    order_id BIGINT,
    buyer_id BIGINT,
    seller_id BIGINT,
    order_date DATE,
    province STRING,
    city STRING,
    amount DECIMAL(18,2),
    PRIMARY KEY (order_id) NOT ENFORCED
) WITH (
    'log.format' = 'arrow'  -- ✅ 查询 SELECT province, SUM(amount) 只需要读两列
);
```

### 4.2 INDEXED（行式索引存储）

**特点**：
- 面向行的 IndexedRow 格式
- 小批量写入更紧凑
- 流式消费更高效
- 不支持列裁剪

**适用示例**：
```sql
-- CDC 同步日志，下游顺序消费
CREATE TABLE cdc.mysql_order (
    order_id BIGINT,
    ...
    PRIMARY KEY (order_id) NOT ENFORCED
) WITH (
    'log.format' = 'indexed'  -- ✅ 小批量 CDC 同步更紧凑
);
```

### 4.3 COMPACTED（紧凑行式存储）

**特点**：
- CompactedRow 格式，专门为主键表优化
- null 值压缩，节省存储空间
- 读取需要更多 CPU 解压

**适用示例**：
```sql
-- 几亿用户维度表，很多字段可空，空间敏感
CREATE TABLE dim.user (
    user_id BIGINT,
    username STRING,
    nickname STRING,
    email STRING,        -- 可空
    phone STRING,        -- 可空
    birthday DATE,       -- 可空
    PRIMARY KEY (user_id) NOT ENFORCED
) WITH (
    'log.format' = 'compacted',  -- ✅ 压缩节省空间
    'kv.format' = 'compacted'
);
```

## 五、虚拟表（Virtual Tables）

### 5.1 什么是虚拟表

虚拟表是一种**不存储数据**的逻辑表，它基于已有的物理表创建，提供不同的访问视角。

主要用途：
- **低延时物化视图**：基于主键表创建流式物化视图，不需要重复存储数据
- **不同分发策略**：同一个数据，按不同分桶策略分发，满足不同查询模式
- **预聚合视图**：基于日志表创建聚合视图，结果实时可见

### 5.2 适用场景

#### （1）多维度分发

同一份数据，按不同维度分桶，加速不同查询：

```sql
-- 原始表：按 order_id 分桶，适合按 order_id 点查
CREATE TABLE orders (
    order_id BIGINT,
    buyer_id BIGINT,
    seller_id BIGINT,
    amount DECIMAL(18,2),
    PRIMARY KEY (order_id) NOT ENFORCED
);

-- 虚拟表：按 buyer_id 重新分桶，加速"查询用户所有订单"
CREATE VIRTUAL TABLE orders_by_buyer
DISTRIBUTED BY (buyer_id) BUCKETS 128
FROM orders;
```

这样，两种查询都能获得好性能：
- 按 order_id 查询 → 走原始表，已经按 order_id 分桶
- 按 buyer_id 查询 → 走虚拟表，已经按 buyer_id 分桶

#### （2）实时物化视图

基于已有表创建聚合视图，结果实时更新：

```sql
-- 原始订单表
CREATE TABLE orders (
    dt DATE,
    province STRING,
    amount DECIMAL(18,2),
    PRIMARY KEY (order_id) NOT ENFORCED
);

-- 实时物化视图：按省份日度统计金额
CREATE VIRTUAL TABLE daily_province_stats
WITH (
    'merge.engine' = 'aggregation',
    'aggregation.aggregations' = 'amount:sum'
)
DISTRIBUTED BY (dt, province) BUCKETS 32
FROM orders;
```

写入 `orders` 后，`daily_province_stats` 实时更新，查询直接返回结果，不需要每次聚合计算。

### 5.3 优势

- **不重复存储**：只维护分发和聚合逻辑，数据仍然只存一份，节省存储空间
- **实时可见**：写入原始表后，虚拟表立即可见，亚秒级新鲜度
- **灵活**：可以随时添加删除虚拟表，不影响原始数据

## 六、分层存储架构

Fluss 原生支持**双层存储架构**：

```
┌─────────────────────────────────────────────┐
│               本地存储                        │
│   - SSD/SATA 磁盘                             │
│   - 保留 N 个最新分段（可配置）                │
│   - 低延迟读写热点数据                        │
└────────────────┬────────────────────────────┘
                 │ 后台异步迁移
                 ▼
┌─────────────────────────────────────────────┐
│               远程存储                        │
│   - 对象存储 S3/OSS/HDFS                     │
│   - 存储所有历史分段                          │
│   - 低成本持久化存储                          │
└─────────────────────────────────────────────┘
```

### 6.1 优势

- **降低成本**：冷数据放在低成本对象存储，节省本地磁盘
- **弹性扩容**：新节点加入不需要迁移所有数据，只需要热点数据
- **更长保留时间**：可以低成本保留几年历史数据

### 6.2 配置示例

```sql
CREATE TABLE user_behavior (
    ...
) WITH (
    'tiered-log.enabled' = 'true',
    'tiered-log.local-segments' = '10',  -- 本地保留 10 个最新分段
    'remote.log.dir' = 's3://bucket/fluss/remote-log/'
);
```

## 七、湖仓集成

Fluss 支持将数据自动下沉到数据湖，实现**流湖一体**：

支持的数据湖：
- [x] Apache Paimon
- [x] Apache Iceberg
- [x] Lance

### 7.1 架构

```
流式写入 → Fluss → 异步定时归并 → 数据湖
                    ↓
最新数据 → Fluss 低延迟读取
历史数据 → 数据湖 大容量分析
```

优势：
- **亚秒级新鲜度**：最新数据在 Fluss 立即可用
- **大容量低成本**：历史数据在数据湖存储
- **统一表抽象**：同一个表名，自动路由到不同存储层
- **对接多种引擎**：Flink/Trino/Spark 都可以查询

### 7.2 配置示例

```sql
CREATE TABLE sales.fact_order (
    ...
) WITH (
    'data-lake.enabled' = 'true',
    'data-lake.catalog-type' = 'paimon',
    'data-lake.warehouse' = 's3://lake/warehouse/'
);
```

## 八、副本机制与高可用

### 8.1 Fluss 支持多副本吗？

| 数据类型 | 支持多副本 | 说明 |
|----------|------------|------|
| **LogTablet（日志数据）** | ✅ 完整支持 | 类似 Kafka，多个副本分布在不同节点 |
| **KvTablet（主键快照）** | ⚠️ 不支持 | 同一个 Bucket 的 Kv 只在 Leader 节点存在，故障时通过快照恢复 |

**为什么 KvTablet 不存多副本？**
- **节省空间**：Kv 数据一般很大，多副本会浪费几倍存储空间
- **恢复可行**：可以从远程快照 + LogStore 重放恢复，不需要多副本存两份

**故障恢复流程（KvTablet）：**
```
原 Leader 宕机
  ↓
Coordinator 分配新节点
  ↓
新节点从远程存储下载最新 Kv 快照
  ↓
从 LogStore 重放快照之后的变更
  ↓
恢复完成，开始服务
```

### 8.2 Leader 是什么粒度？

**答案：每个 Bucket（分桶）一个 Leader**，既不是表级别，也不是节点级别。

```
表 orders (128 个 Bucket)
  ├─ Bucket 0 → Leader 在 TabletServer A
  ├─ Bucket 1 → Leader 在 TabletServer B
  ├─ Bucket 2 → Leader 在 TabletServer A
  ├─ Bucket 3 → Leader 在 TabletServer C
  └─ ...
```

一个 TabletServer 可以同时：
- 是一些 Bucket 的 Leader
- 是另一些 Bucket 的 Follower

### 8.3 Leader 选举

#### （1）新建表初始 Leader 选择

Coordinator 在创建表分配副本时，会追求**Leader 数量在各个节点均衡**：
```
对于每个新建 Bucket：
  ↓
 从它的 N 个副本中
  ↓
 选择「当前已有 Leader 数量最少」的节点作为初始 Leader
  ↓
 如果数量相同，选节点 ID 较小的
```

这样可以保证新建表后，整个集群各个节点的 Leader 负载依然均衡。均衡阈值：允许最大 Leader 数不超过平均的 1.1 倍。

#### （2）原 Leader 故障后的重选

Fluss 有三种场景对应三种选举策略：

| 场景 | 策略 |
|------|------|
| **原 Leader 宕机** | 默认选举：遍历副本分配列表，选第一个**存活且在 ISR 中**的副本 |
| **节点受控下线** | 受控下线选举：排除要下线的节点，选第一个存活且在 ISR 中的副本 |
| **负载重均衡** | 重分配选举：在新分配列表中，选第一个存活且在 ISR 中的副本 |

核心原则：**只从 ISR 中选**，保证选出的 Leader 数据是完整的。

---

## 九、ISR 与 HW 机制

### 9.1 ISR 是什么？

**ISR = In-Sync Replicas（同步中副本）**，它是**所有副本**的子集，只包含「跟得上 Leader 进度」的副本。

```
所有副本 = ISR（同步中） + OSR（Out-of-Sync，不同步）
```

| 状态 | 说明 | 能选 Leader吗 |
|------|------|-------------|
| ISR | 节点存活，复制进度跟得上 Leader | ✅ 可以 |
| OSR | 节点掉线，或者复制落后太多 | ❌ 不能 |

**进出 ISR 规则：**
- Follower 复制落后不超过阈值 → 保持在 ISR
- Follower 超过 `replica.lag.time.max.ms` 没拉取 → 移出 ISR
- Follower 恢复，追赶上 Leader 进度 → 重新加入 ISR

### 9.2 HW（High Watermark 高水位线）是什么？

**HW = 所有 ISR 副本都已经成功复制完成**的最大 offset。消费者**只能消费 HW 以下**的数据。

```
  ◄─── 已经提交，可以消费 ───◄  HW
                          HW ►─── 还在复制，不能消费 ───►
```

**举个例子：**

- 复制因子 = 3，ISR = [A(Leader), B, C]
- Leader 写到 offset 5，B 复制到 5，C 只复制到 3
- HW = **min(5, 5, 3) = 3**
- 消费者只能看到 offset < 3 的数据

当 C 追赶到 5 之后，HW 推进到 5，消费者就能看到所有数据了。

**HW 的作用：**
1. **保证一致性**：消费者只能读到已经被所有 ISR 副本确认的数据
2. **故障不丢数据**：就算 Leader 宕机，新 Leader 也有所有 HW 以下的数据
3. **消费者进度**：重启后从 HW 开始消费，不会丢也不会重复太多

### 9.3 HW 在 Fluss 的存储

每个 Bucket 的 HW 会持久化到本地磁盘：
```
$data-dir/high-watermark-checkpoint
```
重启后直接恢复，不需要重新计算。

---

## 十、Arrow 格式使用问答

### 10.1 LogStore 是基于 Arrow 的列式存储吗？

**不一定，取决于配置：**

```sql
CREATE TABLE ... WITH (
  'log.format' = 'arrow'  -- ← 这个配置决定
);
```

| 配置 | 存储格式 | 类型 |
|------|----------|------|
| `arrow` | Apache Arrow 列式 | ✅ 列式存储（**默认就是这个**） |
| `indexed` | IndexedRow 行式 | ❌ 行式 |
| `compacted` | CompactedRow 紧凑行 | ❌ 行式 |

Fluss 默认就是 `arrow`，因为主打实时分析，列式存储更适合分析。

### 10.2 KvStore 是基于 Arrow 的吗？

**❌ 不是**。KvStore 底层是 RocksDB：
- Key：主键字节
- Value：Fluss 自身行格式（IndexedRow 或 CompactedRow）序列化后的二进制
- 不使用 Arrow 格式，因为 RocksDB 是 KV 存储，随机点查不需要列式

### 10.3 Kv 的 WAL 会用 Arrow 吗？

**✅ 会的**。主键表的变更日志（WAL）写在 LogStore 里：
- 如果 `log.format = arrow` → WAL 就是 Arrow 列式
- 如果 `log.format = indexed/compacted` → WAL 就是对应的行格式

---

## 十一、总结：Fluss 存储设计要点

1. **借鉴 Kafka**：LogStore 充分借鉴 Kafka 分段稀疏索引设计，经过工业界验证
2. **Log + KV 一体化**：主键表既支持点查，又保留完整变更日志供 CDC 消费
3. **灵活的合并引擎**：四种合并引擎满足不同业务合并需求
4. **多种日志格式**：ARROW/INDEXED/COMPACTED 面向不同 workload 优化
5. **原生分层存储**：热冷分离，平衡性能与成本
6. **流湖一体原生集成**：实时数据在 Fluss，历史数据在数据湖
7. **成熟的高可用机制**：Leader 均衡选举、ISR 同步、HW 水位线，思路同 Kafka

---

## 参考资料

- Fluss 官方网站：https://fluss.apache.org/
- Fluss GitHub：https://github.com/apache/fluss
- Kafka 日志结构：https://kafka.apache.org/documentation/#architecture