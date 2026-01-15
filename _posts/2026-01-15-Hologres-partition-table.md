---
layout:     post
title:      "Hologres的分区表为什么不一样"
subtitle:   "SQL Boy心得分享"
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

1. RANGE 分区（范围分区）
适用场景：时间序列数据（日志、事件、监控）、数值区间（如用户 ID 段）。
语法示例:
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
2. LIST 分区（枚举分区）
适用场景：离散枚举值，如 region IN ('cn', 'us', 'eu')、tenant_id、status 等。
语法示例：
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
3. HASH 分区（哈希分区）
适用场景：高基数字段（如 user_id、order_id），用于打散数据分布，避免热点。
语法示例：
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



## 结语

可以用JOIN拉宽，但要学会利用SQL临时表来创建临时结果集，尽量减少IO。在SQL Boy眼中，SQL的优化永无止境。

> 优化不是追求复杂，而是追求简洁高效。好的SQL应该像一首诗，简洁却有力。
