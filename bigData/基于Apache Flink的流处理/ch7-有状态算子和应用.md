# 第七章 - 有状态算子和应用

## 实现有状态函数

### 在 RunningContext 中声明键值分区状态

每个键值，Flink 都会维护一个状态实例。键值分区状态看上去像是一个分布式键值映射，每个函数并行实例都会负责一部分键值并维护相应的状态实例。

键值分区状态只能作用于 KeyedStream 上，可以通过 DataStream.keyBy() 方法获取。

Flink 为键值分区状态提供了很多原语（primitive），它定义了单个键值对的状态结构。Flink 目前支持以下原语：

* ValueState[T] 用于保存类型为 T 的单个值。利用 ValueState.value() 获取状态，通过 ValueState.update(value:T) 更新状态。

* ListState[T] 保存类型为 T 的元素列表。可以看作是 List，静态方法 add()、addAll()、get() 等。

* MapState[K, V] 保存键值映射，类似 Java 的 Map 方法。

* ReducingState[T] 提供了和 ListState 相同的方法，但是它的 add() 方法会返回一个聚合后的值。

* AggregatingState[I, O] 和 ReducingState 行为类型。但它使用 AggregateFunction 来聚合内部的值。


### 通过 ListCheckpointed 接口实现算子列表状态

同一算子并行任务在处理任何事件时都可以访问相同的状态。

在两条数据流上应用带有广播状态的函数需要三个步骤：

1. 调用 DataStream.broadcast() 方法创建一个 BroadcastStream 并提供一个或多个 MapStateDescriptor 对象。
2. 将 BroadcastStream 和一个 DataStream 或 KeyedStream 联结起来。必须将 BroadcastStream 作为参数传给 connect() 方法。
3. 在联结后的数据流上应用一个函数。根据另一条流是否已经按键值分区，比如 KeyedBroadcastProcessFunction 或 BroadcastProcessFunction。

### 使用 CheckpointedFunction 接口

CheckpointedFunction 是用于指定有状态函数的最底层接口。它提供流用于注册和维护键值分区状态以算子状态的钩子函数（hook），同时也是唯一支持使用算子
联合列表状态（UnionListState）。

CheckpointedFunction 接口定义流两个方法：initializeState() 和 snapshotState() 。initializeState() 方法在创建函数的并行实例时被调用。
snapshotState() 方法会在生成检查点之前调用，它需要接收 FunctionSnapshotContext() 对象作为参数。从 FunctionSnapshotContext 中我们可以
获取检查点编号以及 JobManager 初始化检查点的时间戳。

### 接收检查点完成通知

频繁同步是分布式系统产生性能瓶颈的主要原因。Flink 的设计旨在减少同步点的数量，其内部的检查点是基于和数据一起流动的屏障来实现的，因此可以避免算子全
局同步。

## 为有状态的应用开启故障恢复

显式启用周期性检查点机制：

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
// 设置检查点周期为 10 秒（单位毫秒）
env.enableCheckpointing(10000L)
```

## 确保有状态应用的可维护性

Flink 利用保存点机制来对应用以及状态进行维护，但它需要在初始状态就指定好两个参数，一个是算子唯一标识，另一个是最大并行度。

### 指定算子唯一标识

该唯一标识会作为元数据和算子的实际状态一起写入保存点。

强烈建议使用 uid() 方法为应用的算子指定唯一标识。

```scala
val state: DataStream[(String, Double, Double)] = keyedSensorData
  .flatMap(new ...Function())
  .uid("TempAlert")
```

### 定义最大并行度

使用 setMaxParallelism() 方法为每个算子指定最大并行度。它定义了算子在对键值状态进行分割时，所能用到的键值组数量。

```scala
// 为应用设置最大并行度
env.setMaxParallelism(512)
// 为此算子设置最大并行度，会覆盖应用级别的数值
val state: DataStream = keyedSensorData.setMaxParallelism(1024)
```

## 有状态应用的性能及鲁棒性

算子和状态的监护会对应用的鲁棒性和性能产生一定影响。比如状态后端的选择、检查点算法的配置以及应用大小等。

### 选择状态后端

状态后端负责存储每个状态实例的本地状态，并在生成检查点时将它们写入远程持久化存储。状态后端是可插拔的，两个应用可以选择不同的状态后端。

Flink 提供了三种状态后端：MemoryStateBackend、FsStateBackend 以及 RocksDBStateBackend：

* **MemoryStateBackend** 将状态以常规对象的方式存储在 TaskManager 进程的 JVM 堆里。如果状态很大，那么所有在此 JVM 中的任务实例都可能由于 
OOM 而终止。生成检查点时，MemoryStateBackend 会将状态发送到 JobManager 并保存在它的堆内存中。如果一旦出现故障，状态就会丢失。所以建议只用于
开发和调试。  
  
* **FsStateBackend** 也会将状态保存在堆内存中。但它不会在创建检查点时将状态存到 JobManager 的 JVM 堆内，而是将它写到远程持久化文件系统。但它
同样会遇到 TaskManager 内存大小的限制，并可能导致垃圾回收停顿问题。
  
* **RocksDBStateBackend** 会将全部状态存到本地 RocksDB 实例中。**RocksDB** 是一个嵌入式键值存储，它可以将数据保存到本地磁盘上。为了从 RocksDB
中读写数据，系统需要进行序列化和反序列化。相较于前两者，它的优点是支持多个 TB 级的应用，缺点是读写性能偏低。

StateBackend 接口是公开的，所以我们可以自定义状态后端。

### 选择状态原语

状态原语（ValueState、ListState 或 MapState）的选择对应用性能会产生**决定性影响**。例如，ValueState 需要在更新和访问时进行完整的序列化和反
序列化。ListState 需要将它所有的列表条目反序列化。MapState 在遍历时，会从状态后端取出所有序列化好的条目。

### 防止状态泄露

流式应用会长时间运行。为了避免资源耗尽，关键要控制算子状态大小。

导致状态增长的一个常见原因是键值状态的键值域不断发生变化。这种情况下，有状态函数接收的值只有一个特定的活跃期。比如会话 id，一段时间活跃后就再也不再
活跃了，过期值没有价值了。该问题一个解决方案就是从状态中删除那些过期的键值。但很多情况下，函数不会记录它是不是该键值的最后一条，所以无法准确移除过期
状态。

所以只有在键值域不变或有界的前提下才能使用这些函数。比如基于数量或基于事件的窗口不受该问题影响。

## 更新有状态应用

当应用从保存点启动时，应用可以通过以下三种方式进行更新：

1. 在不对已有状态进行更改或删除的前提下更新或扩展应用逻辑。
2. 从应用中移除某个状态。
3. 通过改变状态原语或数据类型来修改已有算子的状态。

### 保持现有状态更新应用

### 从应用中删除状态

新版本从旧版本的保存点启动时，保存点中的部分状态将无法映射到重启后的应用中。

### 修改算子的状态

修改已有算子状态比较复杂。两种办法对状态进行修改：

* 通过更改状态的数据类型，例如将 ValueState[Int] 改为 ValueState[Double]
* 通过更改状态原语类型，例如将 ValueState[List[String]] 改为 ValueState[String]

### 可查询式状态

很多应用需要将它们的结果与其他应用分享。Flink 提供来可查询状态（queryable state）功能。

### 可查询式状态服务的架构及启用方式

Flink 可查询式状态服务包含三个进程：

* QueryableStateClient 用于外部系统提交查询及获取结果。
* QueryableStateClientProxy 用于接收并响应客户端请求。
* QueryableStateServer 用于处理客户端代理的请求。状态服务也需要运行在每个 TaskManager 上。它从状态后端取得状态，并将其返回给客户端代理。

### 对外暴露可查询式状态

实现：定义一个具有键值分区状态的函数，然后在获取状态引用之前调用 StateDescriptor 的 setQueryable(String) 方法。

### 从外部系统查询状态

所有基于 JVM 的应用都可以使用 QueryableStateClient 对运行中 Flink 的可查询式状态进行查询。这个类由 flink-queryable-state-client-java 
依赖提供，可以用以下方式添加：

```xml
<dependency>
    <groupid>org.apache.flink</groupid>
    <artifactid>flink-queryable-state-client-java_2.12</artifactid>
    <version>1.7.1</version>
</dependency>
```

为了初始化 QueryableStateClient，你需要提供一个 TaskManager 的主机名和客户端代理的监听端口。客户端代理的默认监听端口是 9067，你可以在
_./conf/flink-conf.yaml_ 文件中对它进行配置：

```scala
val client: QueryableStateClient = new QueryableStateClient(tmHostname, proxyPort)
```

之后可以调用 getKvState() 方法来查询应用状态。