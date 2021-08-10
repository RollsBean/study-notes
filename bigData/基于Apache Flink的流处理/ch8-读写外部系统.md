# 第八章 - 读写外部系统

## 应用的一致性保障

为了给应用程序提供精确一次的状态一致性保证，应用程序的每个源连接器都需要能够将其读取位置设置为以前的检查点位置。在采取检查点时，数据源算子
将保存其读取位置，并在故障恢复期间基于这些位置进行恢复。

为了实现**端到端精确一次保证**，连接器可以使用这两种技术：幂等性写和事务性写

### 幂等性写

幂等运算可以执行多次，但是最终的结果是一样的。例如，重复地将相同的键值对插入到 Hashmap 中是幂等操作。另一方面，追加操作不是幂等操作，因为多次追加一
个元素会导致结果多次变化。依赖幂等性的接收器来实现精确一次的应用程序必须保证重放时覆盖以前写的结果。

### 事务性写

只将这些结果写入外部接收系统。这类似于数据库的事务，只有提交时才真正修改数据。因此，它也增加了延迟，因为只有在检查点完成后结果才是可见的。

Flink 提供了两个功能模块来实现事务性的接收端连接器，一个通用的 write-ahead-log (WAL) 接收器和一个 two-phase-commit(2PC) 接收器。WAL sink 将所
有结果记录写入应用程序状态，并在接收到完成检查点的通知后将它们发送到 sink 系统。2PC sink 需要一个提供事务支持或公开构建块以模拟事务的接收器系统。对
于每个检查点，接收器启动一个事务并将所有接收到的记录附加到事务中，将它们写入接收器系统而不提交它们。

## 内置连接器

Apache Flink 提供连接器，用于与各种存储系统进行数据读写交互。

### Apache Kafka 数据源连接器

依赖：

```xml
<dependency>
   <groupId>org.apache.flink</groupId>
   <artifactId>flink-connector-kafka_2.12</artifactId>
   <version>1.7.1</version>
</dependency>
```

Flink Kafka 连接器并行地接收事件流，每个并行源任务可以从一个或多个分区读取数据。任务跟踪每个分区的当前读取 offset，并将其包含到检查点数据中。
从失败中恢复时，将恢复 offset，并且源实例将继续从检查点 offset 读取数据。Flink Kafka 连接器不依赖 Kafka 自己的 offset 跟踪机制，该机制基于
所谓的消费 者组。图 8-1 显示了对数据源实例的分区分配。

![Flink read kafka](../../image/bigData/基于Apache%20Flink的流处理/Flink%20read%20kafka.jpeg)


#### Kafka Consumer

Flink 的 Kafka consumer 称为 FlinkKafkaConsumer。它提供对一个或多个 Kafka topics 的访问。

构造函数接受以下参数：

1. Topic 名称或者名称列表
2. 用于反序列化 Kafka 数据的 DeserializationSchema 或者 KafkaDeserializationSchema
3. Kafka 消费者的属性。需要以下属性：
    * “bootstrap.servers”（以逗号分隔的 Kafka broker 列表）
    * “group.id” 消费组 ID

```scala
val properties = new Properties()
properties.setProperty("bootstrap.servers", "localhost:9092")
properties.setProperty("group.id", "test")
val stream = env
    .addSource(new FlinkKafkaConsumer[String]("topic", new SimpleStringSchema(), properties))
```

{% info 提示 %} 请注意，如果一个分区处于不活动状态，则源实例的水位线将不起作用。因此，一个不活动的分区会导致整个应用程序停顿，因为应用程序的水位
线不可用。

#### 配置 Kafka Consumer 开始消费的位置

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment()

val myConsumer = new FlinkKafkaConsumer[String](...)
myConsumer.setStartFromEarliest()      // 尽可能从最早的记录开始
myConsumer.setStartFromLatest()        // 从最新的记录开始
myConsumer.setStartFromTimestamp(...)  // 从指定的时间开始（毫秒）
myConsumer.setStartFromGroupOffsets()  // 默认的方法

val stream = env.addSource(myConsumer)
...
```

* `setStartFromGroupOffsets（默认方法）`：从 Kafka brokers 中的 consumer 组（consumer 属性中的 group.id 设置）提交的偏移量中开始读
  取分区。 如果找不到分区的偏移量，那么将会使用配置中的 `auto.offset.reset` 设置。
* `setStartFromEarliest()` 或者 `setStartFromLatest()`：从最早或者最新的记录开始消费，在这些模式下，Kafka 中的 committed offset 将
  被忽略，不会用作起始位置。
* `setStartFromTimestamp(long)`：从指定的时间戳开始。对于每个分区，其时间戳大于或等于指定时间戳的记录将用作起始位置。如果一个分区的最新记录早
  于指定的时间戳，则只从最新记录读取该分区数据。在这种模式下，Kafka 中的已提交 offset 将被忽略，不会用作起始位置。

#### Kafka Consumer 提交 Offset 的行为配置

配置 offset 提交行为的方法是否相同，取决于是否为 job 启用了 checkpointing。

* 禁用 Checkpointing： 如果禁用了 checkpointing，则 Flink Kafka Consumer 依赖于内部使用的 Kafka client 自动定期 offset 提交功能。
  因此，要禁用或启用 offset 的提交，只需将 `enable.auto.commit` 或者 `auto.commit.interval.ms` 的Key 值设置为提供的 Properties 配置中的
  适当值。


* 启用 Checkpointing： 如果启用了 checkpointing，那么当 checkpointing 完成时，Flink Kafka Consumer 将提交的 offset 存储在
  checkpoint 状态中。 这确保 Kafka broker 中提交的 offset 与 checkpoint 状态中的 offset 一致。 用户可以通过调用 consumer 上的
  `setCommitOffsetsOnCheckpoints(boolean)` 方法来禁用或启用 offset 的提交(默认情况下，这个值是 true )。 注意，在这个场景中，Properties
  中的自动定期 offset 提交设置会被完全忽略。

### Apache Kafka sink 连接器

Flink Kafka Producer 被称为 FlinkKafkaProducer。它允许将消息流写入一个或多个 Kafka topic。

构造器接收下列参数：

1. 事件被写入的默认输出 topic
2. 序列化数据写入 Kafka 的 SerializationSchema / KafkaSerializationSchema
3. Kafka client 的 Properties。下列 property 是必须的：
   * “bootstrap.servers” （逗号分隔 Kafka broker 列表）
4. 容错语义

```scala
val stream: DataStream[String] = ...

val properties = new Properties
properties.setProperty("bootstrap.servers", "localhost:9092")

val myProducer = new FlinkKafkaProducer[String](
        "my-topic",               // 目标 topic
        new SimpleStringSchema(), // 序列化 schema
        properties,               // producer 配置
        FlinkKafkaProducer.Semantic.EXACTLY_ONCE) // 容错

stream.addSink(myProducer)
```

#### Kafka Producer 和容错

启用 Flink 的 checkpointing 后，FlinkKafkaProducer 可以提供精确一次的语义保证。

除了启用 Flink 的 checkpointing，你也可以通过将适当的 `semantic` 参数传递给 FlinkKafkaProducer 来选择三种不同的操作模式：

* `Semantic.NONE`：Flink 不会有任何语义的保证，产生的记录可能会丢失或重复。
* `Semantic.AT_LEAST_ONCE`（默认设置）：可以保证不会丢失任何记录（但是记录可能会重复）
* `Semantic.EXACTLY_ONCE`：使用 Kafka **事务**提供精确一次语义。无论何时，在使用事务写入 Kafka 时，都要记得为所有消费 Kafka 消息的应用程序设置所
  需的 `isolation.level`（`read_committed` 或 `read_uncommitted` - 后者是默认值）。

`Semantic.EXACTLY_ONCE` 模式依赖于事务提交的能力。它可能会造成数据丢失，Flink 重启，但是事务在重启完成之前就超时，此时，事务就会回滚。

默认情况下，Kafka broker 将 `transaction.max.timeout.ms` 设置为 15 分钟。但是 FlinkKafkaProducer 默认事务超时时间是 1 小时，但是这个
参数不允许比 FlinkKafkaProducer 的短，所以如果使用这个模式，还需要修改 Kafka broker 的事务超时时间。

### 文件系统数据源连接器

结合 Apache Parquet 或 Apache ORC 这些高级文件格式，文件系统可以有效地为分析查询引擎，如 Apache Hive、Apache Impala 或 Presto 等提供
服务。因此，文件系统通常用于“连接”流和批处理应用程序。

```scala
val lineReader = new TextInputFormat(null)
val lineStream: DataStream[String] = env.readFile[String](
                                    lineReader, // FileInputFormat 负责读取文件内容
                                    "hdfs:///path/to/my/data", // 读取的路径
                                    FileProcessingMode.PROCESS_CONTINUOUSLY, // 处理模式
                                    30000L) // 时间间隔
```

### 文件系统 sink 连接器

Flink 的 StreamingFileSink 连接器也包含在 flink-streaming-java 模块中。因此，不需要向构建文件添加依赖项。

```scala
import org.apache.flink.api.common.serialization.SimpleStringEncoder
import org.apache.flink.core.fs.Path
import org.apache.flink.streaming.api.functions.sink.filesystem.StreamingFileSink
import org.apache.flink.streaming.api.functions.sink.filesystem.rollingpolicies.DefaultRollingPolicy

val input: DataStream[String] = ...

val sink: StreamingFileSink[String] = StreamingFileSink
    .forRowFormat(new Path(outputPath), new SimpleStringEncoder[String]("UTF-8"))
    .withRollingPolicy(
        DefaultRollingPolicy.builder()
            .withRolloverInterval(TimeUnit.MINUTES.toMillis(15))
            .withInactivityInterval(TimeUnit.MINUTES.toMillis(5))
            .withMaxPartSize(1024 * 1024 * 1024)
            .build())
    .build()

input.addSink(sink)
```

Streaming File Sink 会将数据写入到桶中。由于输入流可能是无界的，因此每个桶中的数据被划分为多个有限大小的文件。如何分桶是可以配置的，默认使用基于
时间的分桶策略，这种策略每个小时创建一个新的桶，桶中包含的文件将记录所有该小时内从流中接收到的数据。

桶目录中的实际输出数据会被划分为多个部分文件（part file），每一个接收桶数据的 Sink Subtask ，至少包含一个部分文件（part file）。额外的部分文
件（part file）将根据滚动策略创建，滚动策略是可以配置的。默认的策略是根据文件大小和超时时间来滚动文件。超时时间指打开文件的最长持续时间，以及文件
关闭前的最长非活动时间。

![streamfilesink bucketing](../../image/bigData/基于Apache%20Flink的流处理/streamfilesink_bucketing.png)

### Apache Cassandra sink 连接器

Apache Cassandra 是一个流行的、可伸缩的和高可用的列存储数据库系统。

Cassandra 的数据模型是基于主键的，所有对 Cassandra 的写入都是基于 upsert 语义。

```scala
val readings: DataStream[(String, Float)] = ???
val sinkBuilder: CassandraSinkBuilder[(String, Float)] =
CassandraSink.addSink(readings)
sinkBuilder
    .setHost("localhost")
    .setQuery(
    "INSERT INTO example.sensors(sensorId, temperature) VALUES (?, ?);")
    .build()
```

带有 WAL 的 Cassandra 接收器连接器是基于 Flink 的 GenericWriteAheadSink operator 实现的。 

## 实现自定义数据源函数

DataStream API 提供了两个接口来实现 source 连接器：

* SourceFunction 和 RichSourceFunction 可以用来定义非并行的 source 连接器，source 跑在单任务上。
* ParallelSourceFunction 和 RichParallelSourceFunction 可以用来定义跑在并行实例上的 source 连接器。

除了并行于非并行的区别，这两种接口完全一样。

SourceFunction 和 ParallelSourceFunction 定义了两种方法：

```scala
/**
 * 启动数据源连接，如果实现了 <code>CheckpointedFunction</code> 接口，则需要在更新状态之前加锁
 */
void run(SourceContext ctx)

/**
 * 取消源的读取，大多数的源在 run() 方法中都使用 while 循环，需要确保在调用 cancel() 之后跳出循环
 */
void cancel();
```

`run()` 方法用来读取或者接收数据然后将数据摄入到 Flink 应用中。根据接收数据的系统，数据可能是推送的也可能是拉取的。Flink 仅仅在特定的线程调用
`run()` 方法一次，通常情况下会是一个**无限循环**来读取或者接收数据并发送数据。任务可以在某个时间点被显式的取消，或者由于流是有限流，当数据被消费完毕时，
任务也会停止。



### 可重置的数据源函数

应用程序只有使用可以重播输出数据的数据源时，才能提供令人满意的一致性保证。如果外部系统暴露了获取和重置读偏移量的 API，那么 source 函数就可以重播源数据。

一个可重置的源函数需要实现 `CheckpointedFunction`接口，还需要能够存储读偏移量和相关的元数据，例如文件的路径，分区的 ID。这些数据将被保存在
list state 或者 union list state 中。

```scala
class ResettableCountSource
    extends SourceFunction[Long] with CheckpointedFunction {

  var isRunning: Boolean = true
  var cnt: Long = _
  var offsetState: ListState[Long] = _

  override def run(ctx: SourceFunction.SourceContext[Long]) = {
    while (isRunning && cnt < Long.MaxValue) {
      // synchronize data emission and checkpoints
      ctx.getCheckpointLock.synchronized {
        cnt += 1
        ctx.collect(cnt)
      }
    }
  }

  override def cancel() = isRunning = false

  override def snapshotState(
    snapshotCtx: FunctionSnapshotContext
  ): Unit = {
    // remove previous cnt
    offsetState.clear()
    // add current cnt
    offsetState.add(cnt)
  }

  override def initializeState(
      initCtx: FunctionInitializationContext): Unit = {
 
    val desc = new ListStateDescriptor[Long](
      "offset", classOf[Long])
    offsetState = initCtx
      .getOperatorStateStore
      .getListState(desc)
    // initialize cnt variable
    val it = offsetState.get()
    cnt = if (null == it || !it.iterator().hasNext) {
      -1L
    } else {
      it.iterator().next()
    }
  }
}
```

### 数据源函数、时间戳及水位线

DataStream API 为分配时间戳和生成水位线提供了两种可选方式。它们可以利用一个专门的 TimestampAssigner 或在数据源函数中完成。

* def collectWithTimestamp(T record, long timestamp): Unit
* def emitWatermark(Watermark watermark): Unit

`collectWithTimestamp()` 用来发出记录和与之关联的时间戳；`emitWatermark()` 用来发出传入的水位线。

## 实现自定义 sink 函数

DataStream API 中，任何运算符或者函数都可以向外部系统发送数据。DataStream API 也提供了 SinkFunction 接口以及对应的 rich 版本
RichSinkFunction 抽象类。SinkFunction 接口提供了一个方法：

```scala
void invoke(IN value, Context ctx)
```

SinkFunction 的 Context 可以访问当前处理时间，当前水位线，以及数据的时间戳。

### 幂等性 sink 连接器

对于大多数应用，SinkFunction 接口足以实现一个幂等性写入的 sink 连接器了。需要以下两个条件：

* 结果数据必须具有确定性的 key，在这个 key 上面幂等性更新才能实现。例如一个计算每分钟每个传感器的平均温度值的程序，确定性的 key 值可以是传感器的
  ID 和每分钟的时间戳。确定性的 key 值，对于在故障恢复的场景下，能够正确的覆盖结果非常的重要。
* 外部系统支持针对每个 key 的更新，例如关系型数据库或者 key-value 存储。

比如，插入数据时先尝试更新，如果数据不存在，再插入。

以 Apache Derby数据库为例，将数据幂等写入 sink 连接器。
```scala
val readings: DataStream[SensorReading] = ...

// write the sensor readings to a Derby table
readings.addSink(new DerbyUpsertSink)

// -----

class DerbyUpsertSink extends RichSinkFunction[SensorReading] {
  var conn: Connection = _
  var insertStmt: PreparedStatement = _
  var updateStmt: PreparedStatement = _

  override def open(parameters: Configuration): Unit = {
    // connect to embedded in-memory Derby
    conn = DriverManager.getConnection(
       "jdbc:derby:memory:flinkExample",
       new Properties())
    // prepare insert and update statements
    insertStmt = conn.prepareStatement(
      "INSERT INTO Temperatures (sensor, temp) VALUES (?, ?)")
    updateStmt = conn.prepareStatement(
      "UPDATE Temperatures SET temp = ? WHERE sensor = ?")
  }

  override def invoke(r: SensorReading, context: Context[_]): Unit = {
    // set parameters for update statement and execute it
    updateStmt.setDouble(1, r.temperature)
    updateStmt.setString(2, r.id)
    updateStmt.execute()
    // execute insert statement
    // if update statement did not update any row
    if (updateStmt.getUpdateCount == 0) {
      // set parameters for insert statement
      insertStmt.setString(1, r.id)
      insertStmt.setDouble(2, r.temperature)
      // execute insert statement
      insertStmt.execute()
    }
  }

  override def close(): Unit = {
    insertStmt.close()
    updateStmt.close()
    conn.close()
  }
}
```

### 事务性 sink 连接器

事务写入 sink 连接器需要和 Flink 的检查点机制集成。

为了简化事务性 sink 的实现，Flink 提供了两个模版用来实现自定义 sink 运算符。这两个模版都实现了 CheckpointListener 接口。CheckpointListener
接口将会从 JobManager 接收到检查点完成的通知。

* `GenericWriteAheadSink` 模版会收集检查点之前的所有的数据，并将数据存储到sink任务的运算符状态中。状态保存到了检查点中，并在任务故障的情况下恢复。
  当任务接收到检查点完成的通知时，任务会将所有的数据写入到外部系统中。
* `TwoPhaseCommitSinkFunction` 模版利用了外部系统的事务特性。对于每一个检查点，任务首先开始一个新的事务，并将接下来所有的数据都写到外部系统的
  当前事务上下文中去。当任务接收到检查点完成的通知时，sink 连接器将会 commit 这个事务。
  
## 异步访问外部系统

有时候，函数里可能还需要从远程数据库获取信息，此时会涉及和外部存储系统交互。一个解决方案是使用 MapFunction，但是它每次请求都会等待查询结果。

Flink 提供的 AsyncFunction 可以有效降低 I/O 调用带来的延迟。该函数能够同时发出多个查询并对其结果进行异步处理。

```scala
trait AsyncFunction[IN, OUT] extends Function {
  def asyncInvoke(input: IN, resultFuture: ResultFuture[OUT]): Unit
}
```

## 总结

* Flink 实现了很多连接器，比如 Hive、Kafka、Cassandra 和文件系统等。
* 通过 `SourceFunction` 和 `SinkFunction` 实现自定义的读写连接器。

