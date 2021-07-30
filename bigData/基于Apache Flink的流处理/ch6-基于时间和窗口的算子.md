# 第六章 基于时间和窗口的算子

## 配置时间特性

时间特性是 StreamExecutionEnvironment 的一个属性，它可以接收以下值：

ProcessingTime  
&nbsp;&nbsp;&nbsp;&nbsp;指定算子根据处理机器的系统时钟决定数据流当前的时间。处理时间基于机器时间触发。

EventTime  
&nbsp;&nbsp;&nbsp;&nbsp;指定算子根据数据自身包含的信息决定当前时间。每个事件时间都带有一个时间戳，系统逻辑时间由水位线来定义。

IngestionTime  
&nbsp;&nbsp;&nbsp;&nbsp;指定每个接收的记录都把在数据源算子的处理时间作为事件时间的时间戳，并自动生成水位线。

### 分配时间戳和生成水位线

时间戳和水位线通过毫秒值指定。水位线用于告知算子不必再等那些时间戳小于或等于水位线的事件。

一般情况下，应该在数据源函数后面立即调用时间戳分配器。

### 水位线、延迟及完整性问题

水位线可用于平衡延迟和结果的完整性。

## 处理函数

Flink 提供来 8 种不同的处理函数，以 KeyedProcessFunction 为例讨论。KeyedProcessFunction 作用于 KeyedStream 上。函数实现了 RichFunction
接口，所以支持 open()、close()、getRuntimeContext() 等方法。还有：

1. processElement(v: IN, ctx: Context, out: Collector[OUT]) 会针对流中的每条记录都调用一次，并返回任意个记录。
2. onTimer(timestamp: Long, ctx: OnTimerContext, out: Collector[OUT]) 是一个回调函数，它在之前注册的计时器触发时被调用。

### 时间服务和计时器

计时器触发时回调用 onTimer() 回调函数。系统对于 processElement() 和 onTimer() 两个方法的调用是同步的，这样可以防止并发访问和操作状态。

### 向副输出发送数据

副输出功能允许从同一函数发出多条数据楼，且类型可以不同。

### CoProcessFunction

## 窗口算子

窗口基于有界区间实现聚合等转换。窗口算子的区间一般基于时间逻辑定义的。

### 定义窗口算子

窗口算子需要指定的两个组件：

1. 窗口分配器（window assigner）：定义如何分配窗口。
2. 窗口函数：作用于 WindowedStream上，用于处理分配到窗口中元素的窗口函数。

```scala
// 定义键值分区窗口算子
stream
  .keyBy(...)
  .window(...) // 指定窗口分配器
  .reduce/aggregate/process(...) // 指定窗口函数
```

### 内置窗口分配器

一旦时间超过了窗口的结束时间就会触发窗口计算。注意：**窗口会随着系统首次为其分配元素而创建，Flink 永远不会对空窗口执行计算。**

Flink 内置窗口分配器的窗口类型是 TimeWindow。该窗口表示两个时间戳之间的时间区间（左闭右开）。

### 在窗口上应用函数

#### 滚动窗口

滚动窗口 TumblingEventTimeWindows 和 TumblingProcessingTimeWindows 分配器接收一个参数：以时间为单位的窗口大小。可以使用 of(Time size) 
方法指定，可以以毫秒、秒、分钟、小时或天数来表示，比如 Time.seconds(1)。也可以通过第二个参数指定 offset。

![tumbling window](../../image/bigData/基于Apache%20Flink的流处理/tumbling%20window.png)

#### 滑动窗口

滑动窗口分配器将元素分配给大小固定且按指定滑动间隔移动的窗口。

![sliding window](../../image/bigData/基于Apache%20Flink的流处理/sliding%20window.png)

```scala
// 事件时间滑动窗口分配器
val slidingAvgTemp = sensorData
  .keyBy(_.id)
  // 每隔 15 分钟创建 1 小时的事件时间窗口
  .window(SlidingProcessingTimeWindows.of(Time.hours(1), Time.minutes(15)))
  .process(new TemperatureAverager)
```

#### 回话窗口

会话窗口将元素放入长度可变且不重叠的窗口中。窗口边界由非活动间隔来定义。

![session window](../../image/bigData/基于Apache%20Flink的流处理/session%20window.png)

```scala
// 会话窗口分配器
val sessionWindows = sensorData
  .keyBy(_.id)
   // 创建 15 分钟间隔的事件事件会话窗口
  .window(EventTimeSessionWindows.withGap(Time.minutes(15)))
  .process(...)
```

除了 `EventTimeSessionWindows` 还有 `ProcessingTimeSessionWindows`。

#### 在窗口上应用函数

窗口函数有两类：

1. **增量聚合函数**。窗口以状态的形式存储某值，根据每个元素对该值进行更新，此类函数一般都十分节省空间。比如 ReduceFunction 和 AggregateFunction。
2. **全量窗口函数**。它会收集窗口内的所有元素。比如 ProcessWindowFunction。

**ReduceFunction**

ReduceFunction 接收两个同类型的值并将它们组合生成一个类型不变的值。ReduceFunction 会对分配给窗口的元素进行增量聚合。

**AggregateFunction**

AggregateFunction 也以增量方式应用于窗口内的元素。

```java
/**
* IN 输入类型
* ACC 累加器类型(内部状态)
* OUT 输出类型
*/
public interface AggregateFunction<IN, ACC, OUT> extends Function, Serializable {
  // 创建一个累加器来启动聚合
  ACC createAccumulator();
  // 向累加器中添加一个输入元素并返回累加器
  ACC add(IN value, ACC accumulator);
  // 根据累加器来返回结果
  OUT getResult(ACC accumulator);
  // 合并两个累加器
  ACC merge(ACC a, ACC b);
}
```

**ProcessWindowFunction**

可以访问窗口内的所有元素并进行计算，比如计算窗口内数据的中值或出现频率最高的值。

```java
public abstract class ProcessWindowFunction<IN, OUT, KEY, W extends Window> 
    extends AbstractRichFunction {
  
  // 对窗口执行计算
  void process(KEY key, Context ctx, Iterable<IN> vals, 
               Collector<OUT> out) throws Exception;
  
  // 当窗口要被删除时，清理一些自定义的状态
  public void clear(Context ctx) throws Exception {}
  
  // Context窗口的上下文
  public abstract class Context implements Serializable {
      
    // 返回窗口元数据
    public abstract W window();

    // 返回当前处理时间
    public abstract long currentProcessingTime();

    // 返回当前事件时间戳
    public abstract long currentWatermark();

    // State accessor for per-window state 每个窗口的状态
    public abstract KeyedStateStore windowState();

    // State accessor for per-key global state 每个键的全局状态
    public abstract KeyedStateStore globalState();

    // Emits a record to the side output identified by the OutputTag.
    // 向OutputTag标识的副输出发送记录  
    public abstract <X> void output(OutputTag<X> outputTag, X value);
  }
}
```


### 自定义窗口算子

DataStream API 对外暴露来自定义窗口算子的接口和方法，你可以实现自己的分配器（assigner）、触发器（trigger）以及移除器（evictor）。

窗口内的状态由以下几部分组成：

窗口内容  
&nbsp;&nbsp;&nbsp;&nbsp;窗口内容包含来分配给窗口的元素，或增量聚合所得的结果。

窗口对象  
&nbsp;&nbsp;&nbsp;&nbsp;WindowAssigner 会返回任意个窗口对象。窗口保存了用于分区窗口的信息，每个窗口对象都有一个结束时间戳。

触发器计时器  
&nbsp;&nbsp;&nbsp;&nbsp;用来在将来某个时间点触发回调。

触发器中的自定义状态  
&nbsp;&nbsp;&nbsp;&nbsp;触发器可以定义自定义状态。

#### 窗口分配器

WindowAssigner 用于决定将到来的元素分配给哪些窗口。

#### 触发器

触发器用于定义何时对窗口进行计算并发出结果。触发条件可以是时间也可以是某些特定的数据条件。

每次调用触发器都会生成一个 TriggerResult，它可以是以下值：

CONTINUE  
&nbsp;&nbsp;&nbsp;&nbsp;什么都不做

FIRE  
&nbsp;&nbsp;&nbsp;&nbsp;如果窗口算子配置了 ProcessWindowFunction，就会调用该函数并发出结果。

PURGE  
&nbsp;&nbsp;&nbsp;&nbsp;完全清除窗口内容，并删除窗口自身及其元数据。同时，调用 ProcessWindowFunction.clear() 方法来清理那些自定义的单个
窗口状态。

FIRE_AND_PURGE  
&nbsp;&nbsp;&nbsp;&nbsp;先进行窗口计算（FIRE），随后删除所有状态及元数据（PURGE）。

#### 移除器

Evictor 是 Flink 窗口机制中的一个可选组件，可用于窗口执行计算前后删除元素。

## 基于时间的双流 Join

### 基于间隔的 Join

对两条流中有相同键值并且时间戳不超过指定间隔的事件进行 Join。

基于间隔的 Join 目前只支持事件时间以及 INNER JOIN 语义。

```scala
imput1
  .keyBy(...)
  .between(<lower-bound>, <upper-bound>) // 相对于 input1 的上下界
  .process(ProcessJoinFunction) // 处理匹配的事件
```

下界和上界分别由负时间间隔和正时间间隔来定义，例如 between(Time.hour(-1), Time.minute(15))。下界值小于上界值。

### 基于窗口的 Join

基于窗口的 Join 需要用到 Flink 中的窗口机制。原理就是将两条输入流中的元素分配到公共窗口中并在窗口完成时进行 Join（或 Cogroup）。

```scala
input1
  .join(input2)
  .where(...) // 为 input1 指定键值属性
  .equalTo(...) // 为 input2 指定键值属性
  .window(...) // 指定 WindowAssigner
  .[trigger(...)] // 选择性地指定 Trigger
  .[evictor(...)] // 选择性地指定 Evictor
  .apply(...) // 指定 JoinFunction
```

两个流根据键值属性分区，公共窗口分配器会将二者的事件映射到公共窗口内。

## 处理迟到数据

如果事件到达算子时窗口分配器为其分配的窗口已经因为算子水位线超过流它的结束时间而计算完毕，那么该事件就被认为是迟到的。

应对迟到事件的方式：

* 简单地将其丢弃。
* 将迟到事件重定向到单独的数据流中。
* 根据迟到事件更新并发出计算结果。

### 丢弃迟到事件

事件时间窗口默认直接丢弃迟到事件。

### 重定向迟到事件

利用副输出将迟到事件重定向到另一个 DataStream，这样可以在后续处理或利用 sink 将其写出。

```scala
 .sideOutputLateData(new OutputTag[SensorReading]("late-readings"))
```

### 基于迟到事件更新结果

可以指定延迟容忍度（allowed lateness），这是一个时间段，配置之后算子会在水位线超过窗口结束时间后不立即删除窗口，而是将窗口继续保留一段时间。这段
时间到达的迟到元素也会交给触发器处理。直到超过了这段时间，窗口才会被删除。