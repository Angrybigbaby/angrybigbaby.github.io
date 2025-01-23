---
layout:     post
title:      "Flink广播的两种形式"
subtitle:   "让我们把它理清楚"
date:       2023-08-02 12:00:00
author:     "Bigbaby"
hidden:	    false
tags:
    - Flink
---

> 最近面试了几个小伙，发现他们对Flink广播的认知十分混乱且流于表面。好像为了应付面试而去背记八股文的小伙越来越多。
> 广播这个东西肯定是好用的，但是怎么用才能发挥它的价值，是我们这次要讨论的东西。

# 什么是广播

Flink不存在跨task通讯（跨task通信的本质是同一TaskManager下跨多个slot通信），因此全局变量不可用，需要考虑广播的形式。广播（broadcast）的概念和Spark很像，指的是把变量或数据加载到其他节点的内存中。
 
这样做的意义在于将不同节点之间的网络通信转化为同节点间的硬件通信。众所周知，硬件通信的成本和稳定性往往高于网络通信，也能够节约带宽和IO。

此外如果当前节点有大于1个Task的情况下，多个Task可以共享当前TaskManager所缓存的广播数据，也就节约了存储空间和通信成本，提升了计算效率。

# 广播的两种形式

### 广播变量（属于DataSet API，已过时）：

从 Flink 1.12开始，DataSet API已被软弃用。

Flink官方建议用Table API和SQL来代替过去的DataSetAPI，这样能够以完全统一的API运行高效的批处理管道。

1. 初始化数据，调用withBroadcastSet()广播数据
 
2. getRuntimeContext().getBroadcastVariable()，获得广播变量
 
3. RichMapFunction中执行获得广播变量的逻辑

```java
/**
 * 演示广播变量
 */
public class BoardcastDemo {
    public static void main(String[] args) throws Exception {
        // 获取env
        ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
        // 准备学生信息数据集
        DataSource<Tuple2<Integer, String>> studentInfoDataSet = env.fromElements(
                Tuple2.of(1, "王大锤"),
                Tuple2.of(2, "潇潇"),
                Tuple2.of(3, "甜甜")
        );
        // 准备分数信息数据集
        DataSource<Tuple3<Integer, String, Integer>> scoreInfoDataSet = env.fromElements(
                Tuple3.of(1, "数据结构", 99),
                Tuple3.of(2, "英语", 100),
                Tuple3.of(3, "C++", 96),
                Tuple3.of(5, "Java", 97),
                Tuple3.of(3, "Scala", 100)
        );

        /*
        广播变量的使用分为两步
        1. 设置它, 需要在执行具体算子的后面链式调用withBroadcastSet()方法
        2. 得到它, 在算子内部getRuntimeContext().getBroadcastVariable(广播变量名)来获取
         */

        // 使用map方法进行转换，在map内部获取广播变量,在map方法的后面链式调用withBroadcastSet方法去设置广播变量
        MapOperator<Tuple3<Integer, String, Integer>, Tuple3<String, String, Integer>> result = scoreInfoDataSet.map(
                new RichMapFunction<Tuple3<Integer, String, Integer>, Tuple3<String, String, Integer>>() {
                    // 定义一个map用来存储从广播变量中取得的学生信息
                    Map<Integer, String> map = new HashMap<Integer, String>();

                    /*
                    open 方法在实例化的开始, 会执行一次
                     */
                    @Override
                    public void open(Configuration parameters) throws Exception {
                        // 在open方法内将广播变量中的学生信息写入到map中
                        List<Tuple2<Integer, String>> broadcastVariable = getRuntimeContext().getBroadcastVariable("student");
                        for (Tuple2<Integer, String> stu : broadcastVariable) {
                            this.map.put(stu.f0, stu.f1);
                        }
                    }

                    @Override
                    public Tuple3<String, String, Integer> map(Tuple3<Integer, String, Integer> value) throws Exception {
                        int stuId = value.f0;
                        String stuName = this.map.getOrDefault(stuId, "未知学生姓名");
                        return Tuple3.of(stuName, value.f1, value.f2);
                    }
                }
        ).withBroadcastSet(studentInfoDataSet, "student");

        result.print();
    }
}
```
可以理解广播就是一个公共的共享变量

将一个数据集广播后，不同的Task都可以在节点上获取到

适用于广播静态数据，即被广播的数据不发生变化

### 广播流（广播状态）：

广播流的本质是将流注册为广播流后，connect()另一个流。

在另一个流中调用process()方法，通过getBroadcastState()方法获取广播状态。

最后通过broadcast()进行广播。

Flink数据流以状态（state）的形式存储当前状态，因此广播流是目的，广播状态是手段，二者称呼不同，但指代的是同一套东西。

```java
// 一个 map descriptor，它描述了用于存储规则名称与规则本身的 map 存储结构
MapstateDescriptor<String, Rule> rulestateDescriptor = new MapstateDescriptor<>(
                                "RulesBroadcaststate",
                                BasicTypeInfO.STRING_TYPE_INFO,
                                TypeInformation.of(new TypeHint<Rule>(){}));

// 广播流，广播规则并且创建 broadcast state
BroadcastStream<Rule> ruleBroadcastStream=rulestream
                                .broadcast(rulestateDescriptor);
```
然后为了关联一个非广播流（keyed 或者 non-keyed）与一个广播流（BroadcastStream），我们可以调用非广播流的方法connect()，并将BroadcastStream当做参数传入。

这个方法的返回参数是BroadcastConnectedStream，具有类型方法process()，传入一个特殊的CoProcessFunction来书写我们的模式识别逻辑。

对于键控（keyed）或者非键控（non-keyed）的非广播流，传入process()的类型有所差异。

    如果流是一个键控（keyed）流，传入KeyedBroadcastProcessFunction类型；
    
    如果流是一个非键控（non-keyed）流，传入BroadcastProcessFunction类型。
    
下面的例子是一个键控流（keyed stream），代码如下：
    
> connect()方法需要由非广播流调用，BroadcastStream作为参数传入。
    
```java
DataStream<String> output = colorPartitionedStream
                 .connect(ruleBroadcastStream)
                 .process(
                     
                     // KeyedBroadcastProcessFunction 中的类型参数表示：
                     //   1. key stream 中的 key 类型
                     //   2. 非广播流中的元素类型
                     //   3. 广播流中的元素类型
                     //   4. 结果的类型，在这里是 string
                     
                     new KeyedBroadcastProcessFunction<Color, Item, Rule, String>() {
                         // 模式匹配逻辑
                     }
                 );
```
在传入的 BroadcastProcessFunction 或 KeyedBroadcastProcessFunction 中，我们需要实现两个方法。processBroadcastElement() 方法负责处理广播流中的元素，processElement() 负责处理非广播流中的元素。 两个子类型定义如下：

```java
public abstract class BroadcastProcessFunction<IN1, IN2, OUT> extends BaseBroadcastProcessFunction {

    public abstract void processElement(IN1 value, ReadOnlyContext ctx, Collector<OUT> out) throws Exception;

    public abstract void processBroadcastElement(IN2 value, Context ctx, Collector<OUT> out) throws Exception;
}
```

```java
public abstract class KeyedBroadcastProcessFunction<KS, IN1, IN2, OUT> {

    public abstract void processElement(IN1 value, ReadOnlyContext ctx, Collector<OUT> out) throws Exception;

    public abstract void processBroadcastElement(IN2 value, Context ctx, Collector<OUT> out) throws Exception;

    public void onTimer(long timestamp, OnTimerContext ctx, Collector<OUT> out) throws Exception;
}
```
	
这里的.processElement()方法，处理的是正常数据流，第一个参数value就是当前到来的流数据；

而.processBroadcastElement()方法就相当于是用来处理广播流的，它的第一个参数value就是广播流中的规则或者配置数据。

两个方法第二个参数都是一个上下文ctx，都可以通过调用.getBroadcastState()方法获取到当前的广播状态；

区别在于.processElement()方法里的上下文是“只读”的（ReadOnly），因此获取到的广播状态也只能读取不能更改；

而.processBroadcastElement()方法里的Context则没有限制，可以根据当前广播流中的数据更新状态。

一个典型的场景是，可用于实时将大表与小表数据进行关联，其中小表数据动态变化。那么可以考虑应用广播流来实现。
