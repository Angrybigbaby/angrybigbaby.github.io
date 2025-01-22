---
layout:     post
title:      "checkpoint与state backends"
date:       2023-03-05 17:41:00
author:     "Bigbaby"
tags:
    - Flink
---

> 老规矩，先下定义。让我们来看看什么是checkpoint，什么是state backends。checkpoint一般翻译为检查点，state backends一般翻译为状态后端。

# 两者的差异
Flink 中的每个方法或算子都能够是有状态的。

状态化的方法在处理单个元素/事件的时候存储数据，让状态成为使各个类型的算子更加精细的重要部分。 

为了让状态容错，Flink 需要为状态添加 checkpoint（检查点）。Checkpoint 使得 Flink 能够恢复状态和在流中的位置，从而向应用提供和无故障执行时一样的语义。

Flink 管理的状态存储在 state backend 中。在启动 CheckPoint 机制时，状态会随着 CheckPoint 而持久化，以防止数据丢失、保障恢复时的一致性。 

状态内部的存储格式、状态在 CheckPoint 时如何持久化以及持久化在哪里均取决于选择的 State Backend。

综上可以简单理解为，checkpoint使得flink可以持久化state。而state backends决定如何实现持久化。

# state backends的分类
Flink 1.13前的状态后端分为：

![image](https://github.com/user-attachments/assets/a731994e-3f31-4f33-9ced-60cb14bab3d3)

其中，RocksDB 是一个 嵌入式本地key/value 内存数据库，和其他的 key/value 一样，先将状态放到内存中，如果内存快满时，则写入到磁盘中。类似Redis内存数据库。
	
状态存储和检查点的创建概念笼统的混在一起。

1.13后(包括1.13)的版本开始,state和checkpoint概念被分开:

State Backend 的概念变窄，只描述状态访问和存储。

![image](https://github.com/user-attachments/assets/c312365e-95a8-461b-afd6-343a38a8c225)

Checkpoint storage，描述的是 Checkpoint 行为，如 Checkpoint 数据是发回给 JM 内存还是上传到远程。
 
![image](https://github.com/user-attachments/assets/597ae92d-47cc-40ce-ad97-595cd74698a3)
