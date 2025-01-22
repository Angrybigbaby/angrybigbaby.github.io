---
layout:     post
title:      "关于Flink end to end exactly once的一些理解"
subtitle:   "关于一致性的理论与实践相结合"
date:       2023-04-15 18:33:00
author:     "Bigbaby"
tags:
    - Flink
---

> 最近一致性这个词已经被玩坏了，成为越来越多业务出身的领导和基础不扎实的架构师喜欢挂在嘴边的概念。
> 在此严正说明：分布式系统的一致性和我们今天要讨论的状态一致性是完全的两码事。
> 那么既然如此，不妨给我们今天要聊的一致性语义来下个定义。
# 关于一致性语义
对于流处理器内部来说，所谓的状态一致性，其实就是所说的计算结果要保证准确。 

一条数据不应该丢失，也不应该重复计算，在遇到故障时可以恢复状态，恢复以后的重新计算，结果应该也是完全正确的

流处理引擎通常为应用程序提供了三种数据处理语义：最多一次、至少一次和精确一次，不同处理语义的宽松定义(一致性由弱到强)：

1. 最多一次：At-most-once：数据可能丢失，但不会重复
   
    当任务故障时，最简单的做法是什么都不干，既不恢复丢失的状态，也不重播丢失的数据。At-most-once 语义的含义是最多处理一次事件。
   
2. 最少一次：At-least-once：数据可能重复，但不会丢失
   
    在大多数的真实应用场景，希望不丢失事件。这种类型的保障称为 at-leastonce，意思是所有的事件都得到了处理，而一些事件还可能被处理多次。
   
3. 精确一次：Exactly-once：数据既不会丢失，也不会重复

    恰好处理一次是最严格的保证，也是最难实现的。恰好处理一次语义不仅仅意味着没有事件丢失，还意味着针对每一个数据，内部状态仅仅更新一次。
   
    Flink的 checkpoint机制和故障恢复机制给Flink内部提供了精确一次的保证。
   
    需要注意的是，所谓精确一次并不是说精确到每个event只执行一次，而是每个event对状态（计算结果）的影响只有一次。

4. End-to-End Exactly-Once

    Flink 在1.4.0 版本引入『exactly-once』并支持『End-to-End Exactly-Once』“端到端的精确一次”语义。
   
    结果的正确性贯穿了整个流处理应用的始终，每一个组件都保证了它自己的一致性。
   
    Flink 应用从 Source 端开始到 Sink 端结束，数据必须经过的起始点和结束点。
   
相比而言，Exactly Once保证所有记录仅影响内部状态一次。而End-To-End Exactly Once保证所有记录仅影响内部和**外部状态**一次。

综上，End-To-End Exactly Once（端到端一致性语义）是Flink级别最高的语义，但他的实现需要上下游组件的配合方可实现。

因此，我们聊到端到端精准一致性语义，需要结合Source和Sink的具体组件来具体分析。

# Flink Datastream如何实现端到端一致性语义
在流式计算引擎中，如果要实现精确一致性语义，有如下三种方式

1. At least once + 去重(需要大量存储,性能开销大)

2. At least once + 幂等性(实现简单,开销较低,依赖存储特性) eg:replace into语句

3. 分布式快照(barrier同步,也就是栅栏对齐,浪费性能)

## Flink内部语义保证
在Flink中,使用检查点checkpoint来保证精确一次性语义,但是checkpoint只能保证内部语义。checkpoint采用分布式快照算法来实现一致性，具体可以参考博主其他文章。

## Source语义保证
在实时且非CDC场景下，FlinkJob最常用的Source以消息队列为主。下面以Kafka为例。

在KafkaSource场景下，消费者是根据offset来持续消费。因此一致性语义的核心是保证任何场景下消费者都能知道上一次的**Offset**

确保offset绝对安全，也就能够确保数据不丢失不重复。而实现绝对安全的方式就是offset持久化，即将offset存储在一种可靠的外部存储中，手动管理offset

在FlinkJob中，KafkaSource可以将偏移量保存到状态后端（具体存储位置由当前Job所配置的状态后端类型决定），如果后续任务出现了故障，恢复的时候可以由连接器重置偏移量，重新消费数据，保证一致性。

Kafka source在checkpoint完成时提交当前的消费offset ，以保证Flink的checkpoint状态和Kafka broker上的提交offset一致。
    
如果未开启checkpoint，Kafka source依赖于Kafka consumer内部的offset定时自动提交逻辑，自动提交功能由enable.auto.commit和auto.commit.interval.ms两个Kafka consumer配置项进行配置。

注意：Kafka source不依赖于broker上提交的offset来恢复失败的作业。提交offset只是为了上报Kafka consumer和消费组的消费进度，以在broker端进行监控。

## Sink语义保证
外部存储介质必须支持**幂等**写入或**事务**写入。其中事务写入的具体实现可分为**预写日志**和**两阶段提交**。

幂等性写入:

    任意多次向一个系统写入数据,只对目标系统产生一次影响
				
事务写入:

    预写日志（Write-Ahead-Log） : 
    
        1. 把结果数据先当成状态保存，然后在收到 checkpoint 完成的通知时，一次性写入 sink 系统；

        2. 简单易于实现，由于数据提前在状态后端中做了缓存，所以无论什么 sink 系统，都能用这种方式；
        
        3. 通用性更强,但是会先写内存 ( 因为状态state只能写入TaskManager堆内存或RocksDB内存数据库 ) ,有丢失风险；
        
        4.  DataStream API 提供了一个模板类：GenericWriteAheadSink，来实现这种事务性 sink； 
        
    两阶段提交（Two-Phase-Commit，2PC） : 
    
        1. 对于每个 checkpoint，sink 任务会启动一个事务，并将接下来所有接收的数据添加到事务里；
        
        2. 然后将这些数据写入外部 sink 系统，但不提交它们 —— 这时只是“预提交”；
        
        3. 当它收到 checkpoint 完成的通知时，它才正式提交事务，实现结果的真正写入；
        
        4. 需要外部sink系统支持事务；
        
        5. Flink 提供了 TwoPhaseCommitSinkFunction 接口。 

> 上述过于抽象，下面结合各组件的FlinkDataStream API具体说明

kafka：

    FlinkKafkaProducer，继承了TwoPhaseCommitSinkFunction，通过kafka事务机制进行两阶段提交（老版API，已被弃用，Flink 1.15 中移除）；

    KafkaSink，在开启checkpoint的情况下所有数据通过在 checkpoint 时提交的事务写入（本质为通过kafka自身的事务机制两阶段提交）（新版API）
```java
DataStream<String> stream = ...;
        
KafkaSink<String> sink = KafkaSink.<String>builder()
        .setBootstrapServers(brokers)
        .setRecordSerializer(KafkaRecordSerializationSchema.builder()
            .setTopic("topic-name")
            .setValueSerializationSchema(new SimpleStringSchema())
            .build()
        )
        .setDeliverGuarantee(DeliveryGuarantee.AT_LEAST_ONCE)
        .build();
        
stream.sinkTo(sink);
```
KafkaSink支持三种不同的语义保证（DeliveryGuarantee）。对于DeliveryGuarantee.AT_LEAST_ONCE和 DeliveryGuarantee.EXACTLY_ONCE，checkpoint必须启用。默认情况下KafkaSink使用 DeliveryGuarantee.NONE。以下详细解释三种语义的配置方法：

DeliveryGuarantee.NONE：不提供任何保证。消息有可能会因Kafka broker的原因发生丢失或因Flink的故障发生重复。

DeliveryGuarantee.AT_LEAST_ONCE: sink在checkpoint时会等待Kafka缓冲区中的数据全部被Kafka producer确认。消息不会因Kafka broker端发生的事件而丢失，但可能会在Flink重启时重复，因为Flink会重新处理旧数据。

DeliveryGuarantee.EXACTLY_ONCE: 该模式下，Kafka sink会将所有数据通过在checkpoint时提交的事务写入。因此，如果consumer只读取已提交的数据（参见Kafka consumer配置isolation.level），在Flink发生重启时不会发生数据重复。然而这会使数据在checkpoint完成时才会可见，因此请按需调整checkpoint的间隔。需确认事务ID的前缀（transactionIdPrefix）对不同的应用是唯一的，以保证不同作业的事务不会互相影响。此外，强烈建议将Kafka的事务超时时间调整至远大于checkpoint最大间隔 + 最大重启时间，否则Kafka对未提交事务的过期处理会导致数据丢失。

MySQL：
    
    使用JDBC sink中的exactly-once模式（因为jdbcSink支持XA标准）

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env
        .fromElements(...)
        .addSink(JdbcSink.sink(
                "insert into books (id, title, author, price, qty) values (?,?,?,?,?)",
                (ps, t) -> {
                    ps.setInt(1, t.id);
                    ps.setString(2, t.title);
                    ps.setString(3, t.author);
                    ps.setDouble(4, t.price);
                    ps.setInt(5, t.qty);
                },
                new JdbcConnectionOptions.JdbcConnectionOptionsBuilder()
                        .withUrl(getDbMetadata().getUrl())
                        .withDriverName(getDbMetadata().getDriverClass())
                        .build()));
env.execute();
```

从 1.13 版本开始，Flink JDBC Sink支持Exactly-Once模式。该实现依赖于JDBC驱动对XA标准的支持。

但诸如PostgreSQL、MySQL等数据库仅允许每次连接一个XA事务。因此需要增加以下API。
```java
JdbcExactlyOnceOptions.builder()
.withTransactionPerConnection(true)
.build();
```

~~写到这不仅感慨哪有什么Java开发，都是API开发而已~~
后面会详细讲一下Flink + Kafka + Hudi场景的一致性实现。
