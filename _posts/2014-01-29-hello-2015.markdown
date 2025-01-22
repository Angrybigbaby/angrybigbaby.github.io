---
layout:     post
title:      "什么是watermark"
subtitle:   " \"终结讨论\""
date:       2023-02-21 12:35:00
author:     "Bigbaby"
catalog: true
tags:
    - Meta
---

> 最近面试很多人，其中在问到“该如何理解Flink的watermark时”，答案五花八门。其实问题的关键在于正确理解乱序和延迟的概念。

## 首先要明确，什么是watermark
watermark可以翻译为水印或水位。Watermark就是给数据再额外的加一个时间列，也就是Watermark是个**时间戳**

允许数据乱序到达，在对应窗口中进行计算（延迟时间很短）

Watermark的作用是用来触发窗口计算的

Watermark = 当前窗口最大事件时间 - 最大允许的**乱序**时间。这样可以保证水位线会一直上升(变大),不会下降

因此watermark的本质就是个时间戳，是Flink给每条数据附加的一种属性

## 其次要搞懂，什么是乱序和延迟
那么既然说到Watermark由窗口时间和最大乱序时间决定，那么如何理解乱序和延迟？

从字面来讲，乱序指的是**同一个窗口**内，数据没有按照原有的顺序到达。而延迟指的是，本该在同一窗口的数据因为各种原因延迟，**在该窗口触发计算**后依然没有到达。

很多人混淆了乱序和延迟的概念。因为从字面上看，两者似乎互为因果。但结合Flink窗口来看，就能简单的区分两种情况的差异

## 最后结合API理解
乱序策略通过调用WatermarkStrategy类中的方法来指定。

```java
public interface WatermarkStrategy<T> 
    extends TimestampAssignerSupplier<T>, WatermarkGeneratorSupplier<T>{

    /**
     * 根据策略实例化一个可分配时间戳的 {@link TimestampAssigner}。
     */
    @Override
    TimestampAssigner<T> createTimestampAssigner(TimestampAssignerSupplier.Context context);

    /**
     * 根据策略实例化一个 watermark 生成器。
     */
    @Override
    WatermarkGenerator<T> createWatermarkGenerator(WatermarkGeneratorSupplier.Context context);
}
```

而延时策略需调用allowed lateness 方法：

```Java
DataStream<T> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .allowedLateness(<time>)
    .<windowed transformation>(<window function>);
```

## 后记

在逐渐熟悉Flink的过程中，会发现有很多抽象的概念难以理解。而且在概念之中，又会夹杂大量细节。因此纯理论学习是件困难且枯燥的事，很容易陷入死记硬背八股文的局面。

马克思曾说过，要将理论与实践相结合。我认为这才是高效学习的本质。放到这个场景下，也就是将概念与代码相结合。除了多翻官方文档以外，要养成阅读源码，理解源码的习惯。

将源码逻辑与文档叙述互相对照理解，将会得出最准确的结论。

祝愿看到这的每个人都能学有所成。
