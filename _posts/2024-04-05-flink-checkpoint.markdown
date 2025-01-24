---
layout:     post
title:      "关于Checkpoint的一些设计"
subtitle:   "核心机制"
date:       2024-04-05
author:     "Bigbaby"
catalog:    true
tags:
    - Flink
    - Checkpoint
---

> Flink 中的每个方法或算子都能够是有状态的。状态化的方法在处理单个元素/事件的时候存储数据，让状态成为使各个类型的算子更加精细的重要部分。
>
> 为了让状态容错，Flink 需要为状态添加 checkpoint（检查点）。Checkpoint 使得 Flink 能够恢复状态和在流中的位置，从而向应用提供和无故障执行时一样的语义。

## State与Checkpoint

Flink管理的状态存储在State Backend中。

有两种State Backend的实现：一种基于RocksDB内嵌Key/Value存储将其工作状态保存在磁盘上的，另一种基于堆的State Backend，将其工作状态保存在Java的堆内存中。

这种基于堆的State Backend有两种类型：FsStateBackend，将其状态快照持久化到分布式文件系统；MemoryStateBackend，它使用JobManager的堆保存状态快照。

Flink使用Chandy-Lamport Algorithm算法的一种变体，称为异步Barrier快照。

当Checkpoint Coordinator（JobManager的一部分）指示TaskManager开始Checkpoint时，它会让所有Sources记录它们的偏移量，并将编号的Checkpoint Barriers插入到它们的流中。这些Barriers流经Job Graph，标注每个Checkpoint前后的流部分。如下图

![image](https://github.com/user-attachments/assets/2403b134-d5bd-4a00-97a6-bb3beef48875)

Checkpoint n将包含每个Operator的State，这些State是对应的operator消费了严格在Checkpoint Barrier n之前的所有事件，并且不包含在此（Checkpoint Barrier n）后的任何事件后而生成的状态。

当Job Graph中的每个Operator接收到Barriers时，它就会记录下其状态。

拥有两个输入流的Operators（例如 CoProcessFunction）会执行Barrier对齐（Barrier Alignment）以便当前快照能够包含消费两个输入流 Barrier 之前（但不超过）的所有Events而产生的状态。如下图：

![image](https://github.com/user-attachments/assets/d9a85e39-60d5-4531-b5fb-c93d5fc2ec33)

在不考虑非对齐检查点（Unaligned Checkpoints）的情况下，综上所述每次Checkpoint都会执行Barrier对齐。对于每个算子而言，Checkpoint所保存的是每个算子完成Barrier对齐的快照（Snapshot）。

假设由于数据倾斜等因素，某一Subtask的处理速度明显滞后于其他线程，将会导致该路Barrier进入算子的时间晚于其他线程。

而此时输入较快的管道数据State会被缓存起来，等待输入较慢的管道数据Barrier对齐。由于输入较快管道数据没被处理，一直积压可能导致OOM或者内存资源耗尽的不稳定问题。由于内存资源耗尽，内存缓冲池被填满，因此将会引起Flink反压。

因此Checkpoint可能会导致反压，而反压也可能会导致Checkpoint延迟。因此合理的设置Checkpoint间隔是必要的。

根据目前生产环境数据量的评估，我们初步确定Checkpoint间隔为2分钟。此外为了确保精准一致性实现，设置Checkpointing模式为精确一次（exactly-once）。

在配置中限定同时运行中Checkpoint的最大数量为1，这是为了限制Checkpointing对资源的占用，提升FlinkJob稳定性。对于Checkpoint的保留策略，以按照保留1小时内的Checkpoint的原则，设定Checkpoint最大保留数（state.checkpoints.num-retained）为6。

对于状态后端的种类，对于小数据量系统考虑使用HashMapStateBackend，其会将数据以对象的形式储存在堆中，内存存储避免了IO，理论上能够获得最佳性能。

而对于大数据量系统考虑使用EmbeddedRocksDBStateBackend，其将数据保存到Flink的内置RocksDB数据库中。这种方式能会发生落盘存储，因此在IO过程中的序列化、反序列化操作可能会影响FlinkJob的吞吐量。但是在大数据量场景往往带来大状态，而HashMapStateBackend在大状态场景可能导致内存占用过高，存在不稳定性。

除了节约内存外，在开启EmbeddedRocksDBStateBackend的基础上可以开启增量检查点优化，进一步缩短checkpoint的完成时间。

> 在这个过程中，最有意思的就是Checkpoint和Backpressure的关系。Checkpoint完成时间过长可能导致Backpressure，同时Backpressure也可能导致Checkpoint完成时间过长或直接失败。真是互为因果的微妙关系。基于这个情况，我特别喜欢把这个知识点拿来当面试题。对于能回答出这个问题的人，在我心中Flink水平就过关了一多半。
