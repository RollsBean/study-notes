# 第五章 DataStream API

## Hello,Flink!

```scala
/** Object that defines the DataStream program in the main() method */
object AverageSensorReadings {

  /** main() defines and executes the DataStream program */
  def main(args: Array[String]) {

    // 设置流式执行环境
    val env = StreamExecutionEnvironment.getExecutionEnvironment

    // 使用事件时间
    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
    // 设置水位线间隔时间
    env.getConfig.setAutoWatermarkInterval(1000L)

    // ingest sensor stream
    val sensorData: DataStream[SensorReading] = env
      // SensorSource generates random temperature readings
      .addSource(new SensorSource)
      // 分配时间戳和水位线
      .assignTimestampsAndWatermarks(new SensorTimeAssigner)

    val avgTemp: DataStream[SensorReading] = sensorData
      // 将华氏温度转为摄氏温度
      .map( r =>
      SensorReading(r.id, r.timestamp, (r.temperature - 32) * (5.0 / 9.0)) )
      // 按照传感器 id 分组
      .keyBy(_.id)
      // 滚动窗口时间间隔 1s
      .timeWindow(Time.seconds(1))
      // 使用 UDF 计算平均温度
      .apply(new TemperatureAverager)

    // 将结果打印到控制台
    avgTemp.print()

    // 执行
    env.execute("Compute average sensor temperature")
  }
}
```

使用常规 Scala 或 Java 方法就可以定义和提交 Flink 程序。构建 Flink 程序步骤如下：

1. 设置执行环境。
2. 从数据源中读取一条或多条流。
3. 流式转换来实现应用逻辑。
4. 将结果输出到一个或多个数据汇中。
5. 执行程序。

### 设置执行环境

我们可以显式指定本地或远程执行环境：

**本地执行**：
```scala
val localEnv = StreamExecutionEnvironment.createLocalEnvironment()
```

**远程执行**：
```scala
val remoteEnv = StreamExecutionEnvironment.createRemoteEnvironment(
  "host",
  1234,
  "path/to/jarFile.jar")
```

### 读取输入流

通过 addSource() 方法创建数据源
```scala
val sensorData: DataStream[SensorReading] = env
      .addSource(new SensorSource)
```

### 应用转换

`map()`、`keyBy()` 转换数据、分组

### 输出结果

将结果写到标准输出

```scala
avgTemp.print()
```

## 执行

Flink 程序都是通过延迟计算的方式执行，只有在调用了 `execute()` 方法时，系统才会触发程序执行。

构建完的计划被装成 JobGraph 并提交到 JobManager 执行。

## 转换操作

转换分为四类：

1. 作用于单个事件的基本转换。
2. 针对相同键值事件的 KeyedStream 转换。
3. 将多条数据流合并为一条或将一条拆分为多条流的转换。
4. 对流中的事件进行重新组织的分发转换。

### 基本转换

#### Map

通过调用 DataStream.map() 进行转换。可以自定义 MapFunction 来自定义转换逻辑（实现 MapFunction 接口）。

```scala
// T: 输入， O：输出
MapFunction[T, O]
  > map(T): O
```

也可以使用 Lambda 函数

```scala
dataStream.map(r => r.id)
```

#### Filter

布尔条件，过滤条件。使用 FilterFunction 或 Lambda 函数实现布尔条件函数。

```scala
// T: 输入
FilterFunction[T, O]
  > filter(T): Boolean
```

#### FlatMap

类似 Map，但是每个输入可以产生任意个输出事件。

```scala
// T: 输入， O：输出
FlatMapFunction[T, O]
  > flatMap(T, Collector[O]): Unit
```

### 基于 KeyedStream 的转换

将事件按键值分配到多条独立的子流中。

> 相同的键值的事件可以访问相同的状态。如果状态存储在内存，需要注意，必须清理不活跃的键值，不然可能导致内存问题。  

#### keyBy

将一个 DataStream 转化为 KeyedStream。

#### 滚动聚合

滚动聚合转换作用于 KeyedStream 上，它会生成一个包含聚合结果（求和、最小值等）的 DataStream。每当有新的事件来，算子都会更新相应的聚合结果。

DataStream API 提供来以下滚动聚合方法：

**sum()**  
&nbsp;&nbsp;&nbsp;&nbsp;滚动求和  

**min()**  
&nbsp;&nbsp;&nbsp;&nbsp;滚动计算最小值  

**max()**
&nbsp;&nbsp;&nbsp;&nbsp;滚动计算最大值  

**minBy()**
&nbsp;&nbsp;&nbsp;&nbsp;滚动计算输入流中迄今为止的最小值，并返回该事件

**maxBy()**
&nbsp;&nbsp;&nbsp;&nbsp;滚动计算输入流中迄今为止的最大值，并返回该事件

**Reduce**

reduce 将 ReduceFunction 应用在 KeyedStream 上，每个到来的事件都和 reduce 结果进行一次组合，从而产生一个新的 DataStream。reduce 转换
不会改变数据类型。

```scala
// T: 元素类型
ReduceFunction[T]
  > reduce(T, T): T
```


### 多流转换

**Union**

DataStream.union() 方法可以合并两条或多条类型相同的 DataStream，生成一个新的 DataStream。

union 执行过程中，两条流的事件以 FIFO 的方式合并，其顺序无法得到保证。此外，union 算子不会对数据去重。

**Connect，coMap，coFlatMap**

connect() 函数将两条流联结，返回一个 ConnectedStreams，它提供了 map() 和 flatMap() 方法，它们分别接收一个 CoMapFunction 和 CoFlatMapFunction
作为参数。

```scala
// IN1: 第一个输入流的类型
// IN2：第二个输入流的类型
CoMapFunction[IN1, IN2, OUT]
  > map1(IN1): OUT
  > map2(IN2): OUT
```

**Split 和 Select**

split 是 union 的逆操作。

DataStream.split() 方法会返回一个 SplitStream 对象，它提供了 select() 方法。

### 分发转换

随机  
&nbsp;&nbsp;&nbsp;&nbsp;利用 DataStream.shuffle() 方法实现随机数据交换策略。它会随机地将记录发往后继算子的并行任务。

轮流  
&nbsp;&nbsp;&nbsp;&nbsp;rebalance() 会以轮流方式均匀分配给后继任务。

重调  
&nbsp;&nbsp;&nbsp;&nbsp;rescale() 也会以轮流方式对事件进行分发。

下图展示了轮流（图a）和重调（图b）的连接模式。
![rebalance&rescale](../../image/bigData/基于Apache%20Flink的流处理/rebalance&rescale.png)

广播  
&nbsp;&nbsp;&nbsp;&nbsp;broadcast() 方法将输入流中国的事件复制并发往所有下游算子的并行任务。

全局  
&nbsp;&nbsp;&nbsp;&nbsp;global() 方法将输入流所有事件发往下游算子的第一个并行任务。所有事件发往同一任务可能会影响程序性能。

自定义  
&nbsp;&nbsp;&nbsp;&nbsp;可以利用 partitionCustom() 方法自己定义分区策略。

## 设置并行度

Flink 应用可以并行执行。算子并行化任务的数目称为该算子的**并行度**。

算子的并行度可以在执行环境级别或单个算子级别进行控制。如果应用在本地执行，并行度会设置为 CPU 的线程数目。

一般，最好将并行度设置为随环境变化的值，这样可以很方便地调整并行度，从而实现扩缩容。

```scala
val parallelism = env.env.getParallelism
```

设置并行度

```scala
env.setParallelism(32)
```

## 类型

Flink 中有个类型提取系统，它可以通过分析函数的输入、输出类型来自动获取类型信息，继而得到相应的序列化器和反序列化器。

### 支持的数据类型

Flink 支持的类型：

* 原始类型。
* Java 和 Scala 元祖
* Scala 样例类。
* POJO（包括 Apache Avro 生成的类）。
* 一些特殊类型。

无法被处理的类型会被当成泛型交给 Kryo 序列化框架进行序列化。

### 为数据类型创建类型信息

Flink 类型系统的核心类是 TypeInformation，它为系统生成序列化器和比较器提供了必要的信息。

### 显式提供类型信息

Flink 的类型提取器会利用反射以及分析函数签名和子类信息的方式，从用户自定义函数中提取正确的输出类型。

## 定义键值和引用字段

### 字段位置

### 字段表达式

### 键值选择器

## 实现函数

### 函数类

### Lambda 函数

### 富函数

富函数的命名规则是以 Rich 开头，后面跟着普通转换函数的名字，如：RichMapFunction 等。富函数生命周期方法：

* open() 是富函数的初始化方法。在任务首次调用转换方法（如 filter 或 map）前调用一次。

* close() 作为函数的终止方法，每个任务最后一次调用转换方法后调用一次。通常用于清理和释放资源。

## 导入外部和 Flink 依赖

为了保持默认依赖的简洁性，对于应用的其他依赖必须显示提供。

两种方法保证在执行应用时可以访问到所有依赖：

1. 将全部依赖打进应用的 JAR 包。这样会生成一个独立但很大的 JAR。
2. 将依赖的 JAR 包放到设置 Flink 的 _./lib_ 目录中。

推荐使用构建 "胖 JAR" 的方式来处理应用依赖。