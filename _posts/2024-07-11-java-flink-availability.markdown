---
layout:     post
title:      "关于Java代码和FlinkJob的高可用设计"
subtitle:   "到底什么是高可用"
date:       2024-07-11
author:     "Bigbaby"
tags:
    - Java
    - Flink
---

> 我们要先搞清楚什么是高可用。高可用性（High Availability，缩写为HA），指系统无中断地执行其功能的能力，代表系统的可用性程度，是进行系统设计时的准则之一。我认为他和所谓的健壮性、鲁棒性（robustness）等概念都是一个意思。讲人话就是，不要让你的程序轻易挂掉。
>
> 那么这里可展开的空间就很大了。我认为概括的来讲分为两点：消除单点故障，实现保证存活机制。

## Java代码的高可用

这里之所以不说Java进程，是因为Flink也算Java进程。但他的高可用是依赖于另一套机制。我们这里只讨论我们自行开发的Java代码。

现阶段代码采用单节点多线程的Java进程方式实现。由于部分业务系统以UDP协议传输数据，这类系统的特殊性在于程序中断后数据无法补推，因此存在数据丢失的风险。

因此引接区的UDP类型系统的要实现高可用。

基于UDP协议易出现丢包的缺点，以及基于高可用优化的思想，在架构上采用双链路改造的思路。

源端将设立第二个数据源，与第一个数据源在一定时间间隔内发送相同的数据。

两个相同的进程将同时接收两个数据源所发送的数据，存入单集群Kafka的两个Topic，或双集群Kafka的某个相同的Topic。

在这种情况下如果某条链路的进程挂掉，另外一路进程的存活依然可以保证数据正常写入Kafka。

此外采用Crontab定时调度Shell脚本的方式来实现程序的健康监控及自动拉起。

在脚本逻辑中，通过传入PID的方式指定保活进程，通过ps命令查看进程当前状态。如果进程挂掉，就根据指定路径及启动参数重新启动进程。

由于Crontab配置时间间隔最小为1分钟，因此目前考虑每分钟执行保活监控1次。后续结合具体需求，可能考虑通过增加脚本的循环执行逻辑，实现秒级的保活。

综上所述，双链路 + 定时脚本的方式解决了单点故障问题，同时实现了进程的自动拉起。这种方式实现了高可用的同时，降低了前期开发成本与后期运维成本，是目前最佳的解决方案。

## FlinkJob的高可用

我们的Flink部署模式为Flink on Yarn。因此实现高可用主要依赖于Hadoop全家桶中的Yarn。

开发流程为，开发好代码后打成jar包，随后上传到Flink客户端运行。

Job提交后会分配TaskManager和JobManager节点。

其中JobManager作为FlinkJob的主节点，承担资源管理及启动TaskMananger的作用。而TaskManager作为从节点，主要承担具体的数据处理工作。

由于Flink在架构设计上可以实现TaskManager的无限重启，且生产环境Flink集群为多节点，TaskManager节点数量也大于一个。因此TaskManager可以自动实现高可用。

所以FlinkJob的高可用主要体现在JobManager的高可用实现上。

Apache Hadoop Yarn是许多数据处理框架中流行的资源提供程序。Flink服务被提交到Yarn的 ResourceManager，后者在Yarn NodeManager管理的机器上生成容器（Container）。

Flink将其JobManager和TaskManager实例化部署到此类容器中。

在Flink on Yarn中，JobManager和ApplicationMaster是在同一个JVM进程中的，在Application模式中这个进程的入口就是YarnApplicationClusterEntrypoint类。

因此YarnApplicationClusterEntrypoint进程在Flink On Yarn中既作为Yarn的ApplicationMaster，又扮演着Flink主节点的角色。

ApplicationMaster运行在Container中，主要作用是向ResourceManager申请资源并和NodeManager协同工作来运行应用的各个任务然后跟踪它们状态及监控各个任务的执行，遇到失败的任务还负责重启它。

因此在Flink On Yarn中，主节点的高可用实现就转变为了ApplicationMaster的高可用实现，而ApplicationMaster的高可用实现依赖于Yarn的高可用机制。YARN 正在负责重新启动失败的 ApplicationMaster（JobManager）。

ApplicationMaster（JobManager）的最大重启次数是通过两个配置参数定义的。

首先Flink的yarn.application-attempts配置默认为2。这个值受到YARN的yarn.resourcemanager.am.max-attempts的限制，它也默认为2。也就是说理论上最多拉起JobManager两次。

此外对于Checkpoint连续失败的情况，会触发FlinkJob重启。对TolerableCheckpointFailureNumber参数设定为5，在五次Checkpoint连续失败后触发FlinkJob重启。

> 这次关于探索JobManager高可用实现的问题，翻了几个小时源码。作为一个逐渐成长的程序员，我在每个阶段所提出的问题都有不同的解决方案。一开始疑问很基础，随便就可以在CSDN搜到。后面逐渐关注到了一些相对深入的细节，国内网站搜索结果很少了，就只能翻墙去外网看帖子。再逐渐的，连Stack OverFlow、Github Issue或官方社区都无法解决我的问题，那就只能手撕源码，自己做自己的老师。我相信各位如果对技术感兴趣，爱钻研，一定会有同样的感受。共勉吧诸位。
