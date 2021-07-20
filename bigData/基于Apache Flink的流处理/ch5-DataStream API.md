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



### 基本转换

### 基于 KeyedStream 的转换

### 多流转换

### 分发转换

## 设置并行度


## 类型

### 支持的数据类型

### 为数据类型创建类型信息

### 显式提供类型信息

## 定义键值和引用字段

### 字段位置

### 字段表达式

### 键值选择器

## 实现函数

### 函数类

### Lambda 函数

### 富函数

## 导入外部和 Flink 依赖