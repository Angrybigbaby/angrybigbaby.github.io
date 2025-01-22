---
layout:     post
title:      "聊聊反压"
subtitle:   "这次有点干"
date:       2023-05-05 12:00:00
author:     "Bigbaby"
tags:
    - Flink
---

> 实时数据开发的反压问题，就像离线数据开发的数据倾斜，Java后端开发的JVM调优，航空界的发动机，篮球运动员的投篮一样，是最有技术含量，最重要，最能体现一个开发的价值，同时也是最复杂，最困难的问题。应该说衡量一个实时数据开发的经验及技术力的level，就体现在对反压的理解以及处理上。
> 那么，什么是反压？

## 反压的定义
反压是在实时数据处理中，数据管道某个节点上游产生数据的速度大于该节点处理数据速度的一种现象。

反压会从该节点向上游传递，一直到数据源，并降低数据源的摄入速度。

这在流数据处理中非常常见，很多场景可以导致反压的出现，比如, GC导致短时间数据积压，数据的波动带来的一段时间内需处理的数据量大增，**甚至是checkpoint本身都可能造成反压**。

## 反压的现象
![image](https://github.com/user-attachments/assets/6e617d35-b323-4a7e-b359-d272378beb94)

在内部，反压根据输出buffers（内存缓冲池）的可用性来进行判断的。 如果一个task没有可用的输出buffers（内存缓冲池），那么这个 task 就被认定是在被反压。相反，如果有可用的输入，则可认定为闲置。

从Flink WebUI上可以直观看到反压的指标，这是判断是否发生反压最直观的方式。

## 数据积压与反压
这是很多人搞不清楚的问题。但如果不把这两个概念区分清楚，会对后续分析产生阻碍。

> 反压的本质是Flink为了缓解数据积压所设计的机制。也就是说数据积压为因，反压为果，二者存在清晰的因果关系。

而数据积压则是发生反压的根本原因。所以我们接下来要讨论的是，数据积压是怎么导致的。

## 数据积压的后果
反压如果不能得到正确地处理，可能会导致 资源被耗尽 或者甚至出现更糟的情况导致数据丢失。

在大数据量场景下，同一时间点，不管是流处理job还是sink，哪怕1秒的卡顿，可能导致上百万条记录的积压。换句话说，source可能会产生一个脉冲，在一秒内数据的生产速度突然翻倍。某个Subtask积压大量数据会导致数据大量堆积在内存，数据丢失风险激增。

一般短时间的反压并不会对实时任务太大影响，如果是持续性的反压就需要注意了，意味着任务本身存在瓶颈，可能导致潜在的不稳定或者数据延迟，尤其是数据量较大的场景下。
反压的影响主要体现在Flink中checkpoint过程上，主要影响两个方面：

    反压出现时，相关数据流阻塞，会使数据管道中数据处理速度变慢，按正常数据量间隔插入的barrier也会被阻塞，进而拉长，checkpoint时间，可能导致checkpoint超时，甚至失败。
    在对齐checkpoint场景中，算子接收多个管道输入，输入较快的管道数据state会被缓存起来，等待输入较慢的管道数据barrier对齐，这样由于输入较快管道数据没被处理，一直积压可能导致OOM或者内存资源耗尽的不稳定问题。

在Flink 1.13及之前版本的barrier需在各subtask对齐后才能进行checkpoint。如果遇到反压，可能会导致subtask无法传递barrier，从而导致无法进行checkpoint，存在数据丢失风险。1.14版本中进行优化，在未对齐的情况下如果有subtask提前执行完算子计算，那么就对所有subtask进行checkpoint。

## 发生数据积压的根本原因
反压简单讲，1.5+版本后的flink反压机制是在接收端input channel缓存不足的情况下向buffer pool申请缓存，在buffer pool缓存耗尽的情况下反馈给上游，切断ResultPartition到Netty的链路。

由于发送端缓存被快速消耗，所以RecordWriter不能再写数据，从而达到反压的效果。

Flink内部自动实现数据流自然降速，而无需担心数据丢失。Flink所获取的最大吞吐量是由pipeline中最慢的组件决定。

Flink1.5+ 版本引入了基于Credit的流控和反压机制，本质上是将TCP的流控机制从传输层提升到了应用层——InputGate和ResultPartition的层级，从而避免传输层造成阻塞。

![image](https://github.com/user-attachments/assets/ba7ccfce-94c0-4d27-83fa-12d0346cbc36)

![image](https://github.com/user-attachments/assets/3d05bdd9-4fc5-49c4-b7f0-724cc2e33d8a)
TaskManager（TM）启动时，会初始化网络缓冲池（NetworkBufferPool）

    默认生成 2048 个内存块（MemorySegment）
    
    网络缓冲池是Task之间共享的
    
Task线程启动时，Flink 会为Task的 Input Gate（IG）和 ResultSubpartition（RS）分别创建一个LocationBufferPool

    LocationBufferPool的内存数量由Flink分配
    
    为了系统更容易应对瞬时压力，内存数量是动态分配的
    
Task线程执行时，Netty接收端接收到数据时，为了将数据保存拷贝到Task中

    Task线程需要向本地缓冲池（LocalBufferPool）申请内存
    
    若本地缓冲池没有可用内存，则继续向网络缓冲池（NetworkBufferPool）申请内存
    
    内存申请成功，则开始从Netty中拷贝数据
    
    若缓冲池已申请的数量达到上限，或网络缓冲池（NetworkerBufferPool）也没有可用内存时，该Task的Netty Channel会暂停读取，上游的发送端会立即响应停止发送，Flink流系统进入反压状态
经过 Task 处理后，由 Task 写入到 RequestPartition （RS）中

    当Task线程写数据到ResultPartition（RS）时，也会向网络缓冲池申请内存
    
    如果没有可用内存块，也会阻塞Task，暂停写入
    
Task处理完毕数据后，会将内存块交还给本地缓冲池（LocalBufferPool）

    如果本地缓冲池申请内存的数量超过池子设置的数量，将内存块回收给 网络缓冲池。如果没超过，会继续留在池子中，减少反复申请开销

由此不难看出，buffer pool耗尽是导致反压的根本原因，也是数据积压的直接体现。数据积压在缓冲区，导致缓冲区满了，上游也就写不进去。那么为什么会发生积压？

是因为**流系统中消息的处理速度跟不上消息的发送速度**。从而导致了消息的堆积，也就是数据积压。

数据积压可能是因为垃圾回收卡顿可能会导致流入的数据快速堆积。

也可能由于一个数据源可能生产数据的速度过快。

## 解决方案探索
> 综上所述，反压的原因可以归类为垃圾回收、线程竞争、数据倾斜、外部数据库性能瓶颈。以下阐述每类问题的解决思路。

1. 垃圾回收

    性能问题常常源自过长的GC时长。这种情况下可以通过打印GC日志，或者使用一些内存/GC分析工具来定位问题。

2. 线程竞争

    CPU/线程瓶颈。
<s>加资源吧</s>

3. 数据倾斜

    1. Source消费不均匀
    
        调整kafkasource的并发度为kafka分区数的整数倍
        
        直接调用shuffle，rebalance或rescale函数，将数据均衡分配
        
    2. key分配不均匀
    
        缩短窗口的大小(将时间调小)
        
        （key之后开窗的情况下）重新设计key，加上随机前缀或后缀
        
        （key之后不开窗的情况下）预聚合：定时器+状态
        
    3. key值为null
    
        会全跑到一个分区里
        
4. 外部数据库性能瓶颈

   1. （用redis作为）旁路缓存，减少查询维表时的连接次数
   
            在代码中使用jedis = RedisUtil.getJedis()获取redis对象，进行查询或存储操作
       
   2. 异步io
       
            继承'RichAsyncFunction' 类，使用Future接口实现异步发送
   
   3. 分布式缓存

            env.registerCachedFile("file:///users","cachedFile");
            getRuntimeContext().getDistributedCache().getFile()
            通过读取hdfs或本地文件注册为缓存，并在上下文对象方法中获取缓存
				
    4. 调整并行度
    
            增大Sink的并行度，即多线程写入。
        
    5. 调整sink buffer缓冲区大小
    
            因为Sink入库操作多为攒批，因此调大缓冲区在大数据量场景可能提升吞吐量。
