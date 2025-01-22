---
layout:       post
title:        "Flink on Yarn三种部署模式的异同"
author:       "Bigbaby"
header-style: text
catalog:      true
tags:
    - Flink
---

> 一篇个人随笔。

flink支持广义上的三种部署模式，即standalone，Flink on K8s以及Flink on Yarn。考虑到生产环境下单一组件独立部署存在资源不可控的情况，企业多以后两者进行部署。而在后两者中，Flink on YARN是推出最早、受众最广的部署模式。Flink on YARN的本质就是：将JobManager和TaskManagers服务运行在YARN Container容器中。

**Session Mode（会话共享模式）**
	预先在Yarn上启动Flink集群
	多个flink Job共享一个集群
	main方法在客户端执行
	优点：不需要每次递交作业申请资源，而是使用已经申请好的资源，
	从而提高执行效率
	缺点:隔离性差,Jobmanager承担所有taskmanager,负载瓶颈,作业完成后资源不会释放
![image](https://github.com/user-attachments/assets/2537cac0-f88e-4d5d-af85-d616856bcbfb)

**Per-Job Mode（Job 分离模式）**
	Job独享资源
	每个FlinkJob启动单独的flink集群
	main方法在客户端执行
	优点:资源隔离充分,作业运行完成，资源会立刻被释放，不会一直占用系统资源
	缺点:每次递交作业都需要申请资源，会影响执行效率，因为申请资源需要消耗时间
![image](https://github.com/user-attachments/assets/b68ac2b3-001d-4c9d-a1ff-f7e495f98c67)

**Application Mode（应用模式）（Flink1.11新特性）**
	main方法在集群中（Jobmanager）运行（比perjob好，节约下载依赖项带宽）
	为每个提交的应用程序创建一个集群，并在应用程序完成时终止（比session更好，更节约资源，和perjob一样）
    优点:节省下载依赖项所需的带宽（以上内容将允许作业提交更加轻量级，因为所需的Flink jar和应用程序jar将由指定的远程位置拾取，而不是由客户端发送到集群。）
![image](https://github.com/user-attachments/assets/5c6a1f14-60a6-4b23-9aa5-1711015b9d0b)
