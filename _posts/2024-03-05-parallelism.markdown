---
layout:     post
title:      "关于Flink并行度的一些思考"
subtitle:   "结合Hudi"
date:       2024-03-05
author:     "Bigbaby"
tags:
  - Flink
---

> 并行计算是大数据思想的重要体现，也是从MapReduce，Spark，到Flink三代计算引擎始终遵循的基础思想。相比于单机数据处理，并行计算可以充分发挥集群优势，用资源换时间的方式提升计算效率。
>
> 对于Flink来说，合理的并行度规划可以提升数据处理效率，提升时效性。

## Source的设计

每个Worker（TaskManager）都是一个JVM进程，可以在单独的线程中执行一个或多个Subtask。

为了控制一个TaskManager中接受多少个Task，就有了所谓的Task Slots（至少一个）。每个Task Slot代表TaskManager中资源的固定子集。例如，具有3个Slot的TaskManager，会将其托管内存1/3用于每个Slot。

分配资源意味着Subtask不会与其他作业的Subtask竞争托管内存，而是具有一定数量的保留托管内存。注意此处没有CPU隔离；当前Slot仅分离Task的托管内存。通过调整Task Slot的数量，用户可以定义Subtask如何互相隔离。

每个TaskManager有一个Slot，这意味着每个Task组都在单独的JVM中运行（例如，可以在单独的容器中启动）。

具有多个Slot意味着更多Subtask共享同一JVM。同一JVM中的Task共享TCP连接（通过多路复用）和心跳信息。它们还可以共享数据集和数据结构，从而减少了每个Task的开销。

默认情况下，Flink允许Subtask共享Slot，即便它们是不同的Task的Subtask，只要是来自于同一作业即可。

结果就是一个Slot可以持有整个作业管道。由于Flink槽共享机制，Flink集群所需的Task Slot和作业中使用的最大并行度恰好一样。由于现阶段项目源端Kafka为单分区配置，因此我们指定流环境并行度为1，该并行度为环境级别。在源端Kafka分区数不提升的情况下，盲目增加Source并行度会导致线程空跑，浪费资源。

## Sink的设计

对于Sink阶段，我们采用FlinkSQL写入Hudi。与DataStream API不同，FlinkSQL仅支持设定环境级别并行度。没有办法对算子并行度做设计。<s>毕竟SQL也没算子的概念</s>

Flink对SQL的支持基于实现了SQL标准的Apache Calcite。Apache Calcite是一个动态数据的管理框架，可以用来构建数据库系统的语法解析模块。

Flink SQL使用并对其扩展以支持SQL语句的解析和验证。对于SQL的执行计划，由Calcite负责解析SQL，以及规划每个阶段的并行度。该并行度对应DataStream API中的算子级别并行度。因此作为Flink用户，不需额外考虑SQL方面的并行度设计。

---  
因此我们给了FlinkJob一个全局并行度。这个并行度需要考虑上游Kafka和下游Hudi。对Kafka而言，Topic分区数将影响并行度设置。对Hudi而言，Bucket数量也应该是并行度整数倍。
