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

```sql
SELECT o1.id, o1.issue_name, ... 
FROM ods_op_support_operation_issue_p_d oi
LEFT JOIN ods_op_support_sys_dict_data_p_d d1 
  ON oi.issue_source = d1.dict_value 
  AND d1.dict_type = 'issue_source'
  AND oi.ds = d1.ds
-- 重复8次，每个JOIN都单独查询
```

这种8次JOIN的写法看似离谱，实则为了将每次JOIN的结果集拉宽，有一定合理性。但依然存在问题：**对同一个表进行了8次全表扫描查询**。

## 优化思路

调整为CTE结构，并优化为子表嵌套查询

```sql
WITH dict AS (
  SELECT * 
  FROM ods_op_support_sys_dict_data_p_d
  WHERE ds = '${bizdate}'
    AND dict_type IN ('issue_source','restart_or_not',...)
)
SELECT oi.issue_id, ...
FROM ods_op_support_operation_issue_p_d oi
LEFT JOIN dict d1 ON oi.issue_source = d1.dict_value AND d1.dict_type = 'issue_source'
```
优化后，WITH子句执行一次查询，获取所有需要的字典数据。结果存储在临时结果集dict中。后续8个JOIN都**基于这个临时结果集**，不再需要访问原始表。

IO次数由8次下降为1次。

## 深入思考

这个问题带给我们两点启示：

### 1. 避免重复查询

数据库查询是昂贵的操作，重复查询相同表会带来巨大开销。当多个JOIN都指向同一张表时，应优先考虑合并查询。

### 2. 临时表的价值

WITH语句创建的临时表在查询过程中只被扫描一次，后续所有JOIN都基于这个临时结果集，大幅减少I/O。

## 结语

可以用JOIN拉宽，但要学会利用SQL临时表来创建临时结果集，尽量减少IO。在SQL Boy眼中，SQL的优化永无止境。

> 优化不是追求复杂，而是追求简洁高效。好的SQL应该像一首诗，简洁却有力。
