---
layout:     post
title:      "一段SQL引发的思考"
subtitle:   "SQL Boy心得分享"
date:       2025-10-11
author:     "Bigbaby"
tags:
    - SQL
---

> 最近在Review代码时，发现了一个很有代表性的问题。

## 问题背景

原始SQL使用了8个LEFT JOIN，每次JOIN都单独查询`ods_op_support_sys_dict_data_p_d`表：

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
