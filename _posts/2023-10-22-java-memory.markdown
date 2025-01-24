---
layout:     post
title:      "关于Java内存队列的一些应用"
date:       2023-10-22
author:     "Bigbaby"
tags:
  - Java
---

## 什么是队列

在Java中，线程队列是一种数据结构，用于在多个线程之间传递数据。

线程队列可以实现生产者-消费者模式，即一个或多个生产者线程向队列中放入数据，一个或多个消费者线程从队列中取出数据。

线程队列可以保证数据的线程安全性，即在多线程的环境下，不会出现数据的丢失或混乱。

## 怎么用队列

在Java多线程应用中，队列的使用率很高，多数生产消费模型的首选数据结构就是队列。

Java提供的线程安全的Queue可以分为阻塞队列和非阻塞队列，其中阻塞队列的典型例子是BlockingQueue，非阻塞队列的典型例子是ConcurrentLinkedQueue，在实际应用中要根据实际需要选用阻塞队列或者非阻塞队列。

其对应BlockingQueue （阻塞算法）和ConcurrentLinkedQueue（非阻塞算法）。

在近期的项目中我选择ConcurrentLinkedQueue，它是一个无锁的并发线程安全的队列。

对比锁机制的实现，使用无锁机制的难点在于要充分考虑线程间的协调。

简单的说就是多个线程对内部数据结构进行访问时，如果其中一个线程执行的中途因为一些原因出现故障，其他的线程能够检测并帮助完成剩下的操作。

这就需要把对数据结构的操作过程精细的划分成多个状态或阶段，考虑每个阶段或状态多线程访问会出现的情况。

ConcurrentLinkedQueue是一种基于链表的非阻塞队列，它使用CAS算法来保证线程安全，性能比阻塞队列高。

它是一个无界队列，可以无限制地向队列中添加元素。

它是一个FIFO（先进先出）的队列，即先添加的元素先被获取。

ConcurrentLinkedQueue可以用于实现高并发的场景，例如多个线程共享一个任务队列。

ConcurrentLinkedQueue有两个volatile的线程共享变量：head，tail。

要保证这个队列的线程安全就是保证对这两个Node的引用的访问（更新，查看）的原子性和可见性，由于volatile本身能够保证可见性，所以就是对其修改的原子性要被保证。

对于对象入队列的处理原则是，如果队列size未达到预设的capacity，则直接入队列；

如果队列size达到预设的capacity，且autoExtend为true，先扩容capacity（不能超过初始capacity的10倍），然后入队列；

如果队列size达到预设的capacity，autoExtend为false或capacity已经达到初始capacity的10倍，如果discardOld为true，则出队一个对象后数据入队列；如果discardOld为false，则数据不入队列，返回false。

## 如何配置队列

    1.	如果队列size未达到预设的capacity，则直接入队列（不同系统的capacity值不同）
    2.	如果队列size达到预设的capacity，且autoExtend为true，先扩容capacity（不超过初始capacity的10倍，即targetCapacity = capacity + initCapacity * 0.5，且targetCapacity < initCapacity*10）
    3.	如果队列size达到预设的capacity，autoExtend为false或capacity已经达到初始capacity的10倍，如果 discardOld为true，则出队一个对象后数据入队列（不同系统的discardOld值不同）
    4.	如果discardOld为false，则数据不入队列，返回false

其中，对于不同业务系统，可以分别设定capacity的值。
