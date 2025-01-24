---
layout:     post
title:      "Hudi场景下的Flink调优"
subtitle:   "还是干货"
date:       2024-06-21
author:     "Bigbaby"
tags:
    - Hudi
    - Flink
---

> 在我的项目中，FlinkJob的调优可分为资源调优，性能调优及Hudi写入调优三种类型。以下分别阐述每种类型的调优思路。

## 资源调优

资源调优方面，核心在于压缩TaskManager及JobManager占用总资源，因此开启槽共享并合理设置Slot数是其中的关键。

Flink经由算子链优化机制，在分布式执行场景将算子的Subtask链接成Task，每个Task由一个线程执行。它减少线程间切换、缓冲的开销，并且减少延迟的同时增加整体吞吐量。

Flink槽（Slot）设计初衷是为了控制TaskManager作为一个JVM进程其中有多少线程，也就是FlinkJob的Subtask。

而槽共享机制允许多个Subtask共享一个槽。

在经过算子链与槽共享优化机制后，TaskManager的每个Slot可以运行一个完整的Task。

通常考虑把Slot个数设定为CPU核心数。Slot不会涉及CPU的隔离，也不会相互竞争托管内存，而是一开始就会分配好固定的托管内存。

对于同样并行度的FlinkJob，单Slot多TaskManager的场景意味着每个Task在单独的JVM虚拟机中运行。而多Slot单TaskManager的场景中多Subtask共享同一JVM虚拟机。

在Flink on Yarn模式中，同一JVM中的Task处于同一个Container中，也就节约了Yarn ApplicationMaster（Flink JobManager）的管理与监控成本。

此外还可以共享数据集和数据结构，从而减少了每个Task的开销。

在这个项目中，Slot数量考虑与环境并行度相等，而环境并行度参考Source Kafka分区数。考虑到Topic后续可能扩充为多分区，因此暂定Slot数量为3.

## 性能调优

性能调优方面，核心在于FlinkJob资源的规划。

对于数仓任务而言利用有限资源发挥最大性能是性能调优的整体目标。

在状态管理的优化上，将默认的HashMap状态后端更改为EmbeddedRocksDB状态后端。后者会在ManagedMemory（托管内存）写满后在硬盘中进行存储，实现节省内存的作用，且能够使用增量检查点。

对于生产环境可能出现的Checkpoint用时过长的问题，引入增量检查点 + 非对齐检查点的模式。

传统的全量检查点是Flink默认的检查点模式。每一个Flink的检查点都保存了程序完整的状态。而增量检查点仅保存过去和现在状态的差异部分。

由于RocksDB支持增量快照，基于RocksDB状态后端的增量检查点可以显著减少Checkpoint中完成快照的耗时。

非对齐检查点主要用于由于解决多并行度场景或多写场景部分Task数据量过大，从而导致Checkpoint完成耗时过长的问题。

其核心是，由于对齐检查点的Barrier对齐机制，导致数据倾斜场景可能由于Task的Barrier延迟，导致其他Task阻塞，甚至可能导致反压。

而非对齐检查点在上述场景不需等待其他Task的Barrier对齐就可以马上开始快照，这可以从很大程度上加快Barrier流经整个DAG的速度，从而降低Checkpoint整体时长。

## Hudi写入调优

Hudi Sink作为FlinkJob与外部数据库交互的环节，写入性能会极大影响FlinkJob整体性能。

因此对Hudi写入性能的优化尤其重要。

在索引环节，将默认的State索引更改为Bucket索引。Bucket 索引通过固定的 Hash 策略，将相同 Key 的数据分配到同一个 FileGroup 中，避免了索引的存储和查询开销。

在实时大数据量场景下，Bucket索引理论上性能优于State索引。然而由于Bucket索引无法扩增Bucket，因此在初期设置合理的Bucket数量是必要的。

由于现阶段系统均为流式写入，因此数据量会逐渐累积。假如Bucket数量设置过少，会导致后期同一Bucket内数据量过大，在同一FileGroup（文件组）中扫描的性能下降；而Bucket设置过多则会引起小文件问题。

从宏观角度考虑，假设数据无限期存储，那么理论上Bucket数量越多则Bucket索引性能越好。此外如果Bucket数量不为写入并行度的整数倍，可能导致写入时发生数据倾斜。

而如果写入并行度与Source并行度不一致，又可能由于发生Shuffle导致Flink性能下降。

对于上游为Kafka数据源的系统，在更改为不同库表对应不同侧输出流的形式后，在写入端就得以避免数据重复的问题。

因此，可以采用Insert写入。Insert可以使写入性能得以提升。因为Upsert会就指定主键对每个FileGroup（文件组）的FileSlice（文件片）进行扫描，将主键相同的数据进行更新。

而Insert写入则不需要进行该过程，从而提升了性能。

对于MOR表而言，由于需要将基本文件和日志合并以产生新的文件片（Compaction机制），而Compaction操作需要额外消耗资源，因此采用异步Compaction的方式。

此外，适当调整Compaction的可用内存，将默认的100MB调整为1GB，规避可能出现的由于资源瓶颈导致的Compaction失败的问题。

> 这里提一嘴加括号的原因，基本上我的博文都会有各种括号的英译中或中译英。主要是平时很多名词喜欢用中文表述，但见的人多了就发现一个词真的会有不同翻译，而不同翻译引起的歧义可能直接导致对这个词背后的概念出现理解偏差。所以为了避免歧义，我会提供官方的英文名称，以及我个人习惯称呼的中文翻译。大家可以对照着看，避免引起歧义。
