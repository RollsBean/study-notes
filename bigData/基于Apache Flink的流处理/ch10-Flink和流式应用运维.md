# 第十章 - Flink和流式应用运维

## 运行并管理流式应用

为了监控主进程、工作进程，Flink 对外公开流以下接口：

1. 用于提交和控制应用的命令行客户端工具。
2. REST API。
3. Flink Web UI。

### 保存点

保存点需要用户或外部服务手动触发。

### 命令行客户端

Flink 命令行提供了启动、停止和管理 Flink 应用的功能。可以通过安装根目录下的命令 _./bin/flink_ 调用它。

#### 启动应用

使用 run 命令来启动应用：
```shell
./bin/flink run ~/myApp.jar
```

上述命令会从 JAR 包中 _META-INF？MANIFEST.MF_ 文件中的 pargram-class 属性所指定的 main() 方法启动应用。客户端会将 JAR 包提交到主进程，
随后再由主进程分发至工作节点。

默认情况下，提交应用后不会立即返回，而是等待它停止。可以使用 -d 参数以**分离模式**提交应用：

```shell
./bin/flink run -d ~/myApp.jar
```

还可以通过 -p 参数指定并行度，如果 JAR 中的 manifest 文件没有指定入口类，可以使用 -c 参数指定它：

```shell
./bin/flink run -c my.app.MainClass ~/myApp.jar
```

#### 列出正在运行的应用

```shell
./bin/flink list -r 
```

#### 生成和清除保存点

使用如下命令为应用生成一个保存点：
```shell
./bin/flink savepoint <jobId> [savepointPath] 
```

如为某作业生成保存点，并将其存到 _hdfs://xxx:50070/savepoints_ 目录中：
```shell
./bin/flink savepoint <jobId> hdfs://xxx:50070/savepoints
```

Flink 不会自动清除保存点，手动删除命令如下：
```shell
./bin/flink savepoint -d <savepointPath> 
```

取消应用：
```shell
./bin/flink cancel <jobId> 
```

#### 通过 REST API 管理应用

Web 服务同时支持 REST API 和 Web UI，该服务器会作为 Dispatcher 进程的一部分来运行。默认情况下两者都使用 8081 端口。例如请求 /overview 查询
集群基本信息：

```shell
curl -X GET http://localhost:8081/v1/overview
```

## 控制任务调度

介绍如何通过调整默认行为以及控制任务链接和作业分配来提高应用的性能。

### 控制任务链接

任务链接指的是将两个或多个蒜子的并行任务融合在一起，从而可以让它们在同一线程中执行。默认会开启，它可以减少网络通信成本，一般可以提高性能。但是如果
需要将负载较重的函数拆开，也可以通过 env.disableOperatorChaining() 禁用任务链接。

### 定义处理槽共享组

Flink 默认将一个完整的程序分片分配到一个处理槽中。Flink 提供了处理槽共享组（slot-sharing group）机制。同一处理槽的算子会在一个处理槽执行。

## 调整检查点及恢复

Flink 提供了一系列用于调整检查点和状态后端的参数。这对于生成环境很重要，配置好可以保证流式应用可靠、稳定地运行。

### 配置检查点

启用检查点后，必须指定生成间隔，通过如下方式生成：

```scala
// 启用间隔为 10 秒的检查点
env.enableCheckpointing(10000)
```

从 StreamExecutionEnvironment 中获取 CheckpointConfig 对检查点进行配置：

```scala
val cpConfig: CheckpointConfig = env.getCheckpointConfig
// 设置为至少一次模式
cpConfig.setCheckpointingMode(CheckpointingMode.AT_LEASTONCE)
```

#### 启用检查点压缩

Flink 支持对检查点和保存点进行压缩。

### 配置状态后端

默认状态后端是 MemoryStateBackend。显示指定状态后端：

```scala
val env: StreamExecutionEnvironment = ???
// 创建并设置状态后端
val stateBackend: StateBackend = ???
env.setStateBackend(stateBackend)
```

比如，使用默认状态后端：

```scala
val backend = new MemoryStateBackend()
```

### 配置故障恢复

故障恢复后，为了赶上数据流的进度，应用处理速度要大于新数据到来的速率。

#### 重启策略

Flink 提供了三种重启策略：

* _fixed-delay_ 以固定间隔、固定次数尝试重启应用。
* _failure-rate_ 配置一定时间内最大的故障次数，比如可以配置最近十分钟内发生故障的次数没超过三次，就一直重启。
* _no-restart_ 不重启应用，让它立即失败。

默认是 十秒内尝试 Integer.MAX_VALUE。

## 监控 Flink 集群和应用

### Flink Web UI

通过 _http://<jobmanager-hostname>:8081_ 地址来访问它。

### 指标系统

Flink Web UI 提供了很多指标，指标类别包括 CPU 利用率、内存使用情况、活动线程数、GC等。

##配置日志行为

FLink 使用 SLF4J，要修改日志，可以修改 _conf/_ 目录中的 _log4j.properties_ 文件。