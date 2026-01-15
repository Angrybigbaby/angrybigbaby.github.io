---
layout:     post
title:      "Hologres的分区表为什么不一样"
subtitle:   "由浅入深，着眼细节"
date:       2026-01-15
author:     "Bigbaby"
tags:
    - SQL
    - PostgreSQL
    - Hologres
---

> Hologres 是一个兼容 PostgreSQL 协议、但内核深度重构的云原生实时数仓引擎。在这一背景下，Hologres的分区特性和PostgreSQL有相似性，但也有很多不同。和常用的Hive、Spark等离线大数据引擎相比，差异更加明显。

## Hologres分区类型

Hive 只有一种分区：基于字符串的静态目录映射。
相比而言，Hologres 提供了三种原生分区策略，每种都深度集成于其列存引擎与分布式执行器中。

Hologres 基于 PostgreSQL 分区语法扩展，目前原生支持以下三种分区策略：
RANGE 分区（范围分区）
LIST 分区（枚举分区）
HASH 分区（哈希分区）

> 注：Hologres 不支持 Hive 那种“多级字符串路径式分区”，也不支持“动态分区自动建目录”。所有分区必须通过 DDL 显式定义。

## 分区类型详解与对比

### 1. RANGE 分区（范围分区）
适用场景：时间序列数据（日志、事件、监控）、数值区间（如用户 ID 段）。

```sql
CREATE TABLE logs (
    ts TIMESTAMPTZ,
    msg TEXT
) PARTITION BY RANGE (ts);

CREATE TABLE logs_202501 PARTITION OF logs
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
```
优点：
查询带范围条件时，分区剪枝极高效；
支持时间函数直接比较（ts > now() - interval '7 days'）；
可配合 自动间隔分区（hg_create_interval_partition）简化运维。

缺点：
数据倾斜风险高（如某天流量暴增）；
必须预先创建分区，否则写入失败。

### 2. LIST 分区（枚举分区）
适用场景：离散枚举值，如 region IN ('cn', 'us', 'eu')、tenant_id、status 等。

```sql
CREATE TABLE user_events (
    region TEXT,
    event TEXT
) PARTITION BY LIST (region);

CREATE TABLE user_events_cn PARTITION OF user_events
    FOR VALUES IN ('cn');

CREATE TABLE user_events_global PARTITION OF user_events
    DEFAULT;  -- 默认分区（可选）
```
优点：
枚举值查询可精准命中单个分区；
适合多租户或地域隔离场景；
支持 DEFAULT 分区兜底未知值。

缺点：
枚举值过多时（>1000），管理成本高；
不适合连续值或高基数字段（如 user_id）。

### 3. HASH 分区（哈希分区）
适用场景：高基数字段（如 user_id、order_id），用于打散数据分布，避免热点。

```sql
CREATE TABLE user_profiles (
    user_id BIGINT,
    name TEXT
) PARTITION BY HASH (user_id);

-- 创建 4 个哈希分区
CREATE TABLE user_profiles_p0 PARTITION OF user_profiles
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE user_profiles_p1 PARTITION OF user_profiles
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);
-- ... p2, p3
```
优点：
天然负载均衡，避免写入/查询热点；
适合点查（WHERE user_id = 12345），可通过哈希函数快速定位分区；
与主键结合，可实现高效 UPSERT。

缺点：
范围查询无法剪枝（如 user_id BETWEEN 1 AND 1000 会扫描所有分区）；
分区数量需提前规划，后期扩容复杂（需重分布数据）。

## 与Hive分区的差异

| 维度 | Hive “分区” | Hologres 分区（RANGE/LIST/HASH） |
|------|-------------|-------------------------------|
| 本质 | 元数据对 HDFS 路径的字符串映射 | 数据库内核管理的物理子表 |
| 类型支持 | 仅“字符串枚举”，无语义 | 强类型 + 三种策略（范围/枚举/哈希） |
| 数据写入 | 写入即自动创建目录（动态分区） | 必须显式 DDL 创建分区，否则报错 |
| 更新能力 | 不支持 UPDATE/DELETE（ACID 表性能差） | 原生支持 UPSERT/DELETE（需主键） |
| 查询优化 | 仅目录级剪枝 | 分区剪枝 + 列存谓词下推 + 向量化执行 |
| 存储模型 | 依赖外部文件格式（Parquet/ORC） | 内置列存，自动压缩、索引、合并 |
| 小文件问题 | 严重，需手动 compaction | 无小文件，写入由引擎自动合并 |
| 实时性 | 批处理，分钟级以上 | 写入秒级可见，支持流式摄入 |

## 如何选择分区策略

| 场景 | 推荐分区类型 |
|------|-------------|
| 时间序列数据（日志、事件） | RANGE（按天/小时） |
| 多租户、地域隔离 | LIST（region/tenant_id） |
| 高并发点查（用户画像、订单） | HASH（user_id/order_id） |
| 混合负载（点查+范围扫描） | 先 HASH 再 RANGE（Hologres 支持多级分区） |

>  注：Hologres 支持多级分区（如 PARTITION BY HASH(user_id), RANGE(event_time)），这是 Hive 完全无法实现的组合能力。开源PostgreSQL多级分区性能较差，而Hologres则具备生产级应用能力。

## 聊点深入的

我们常用的分区键是时间字段。那么为什么时间分区更适合range而不是list？首次在 Hologres 建分区表时，这是我在思考的一个问题。

所以以时间字段为例，我们来从几个角度分析这个问题。

### 查询效率

假设查询：
```sql
SELECT * FROM logs WHERE event_time >= '2025-01-10' AND event_time < '2025-01-20';
```
RANGE 分区：
优化器通过区间比较，直接定位到覆盖 [2025-01-10, 2025-01-20) 的分区（可能1个或2个），其余全部跳过。

LIST 分区（按天建分区）：
即使你为每一天建一个 LIST 分区（如 p_20250101, p_20250102, ...），优化器也无法自动推导出“10号到19号”对应哪些分区。
它必须：
  1. 解析谓词；
  2. 枚举所有可能的日期值；
  3. 逐个检查是否在某个 LIST 分区的 IN (...) 列表中；
  4. 最坏情况下仍需扫描多个分区元数据。



> 优化不是追求复杂，而是追求简洁高效。好的SQL应该像一首诗，简洁却有力。
