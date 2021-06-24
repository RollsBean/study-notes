# 第三章 Apache Flink 架构

## 系统架构

Flink 是一个用于状态化并行流处理的分布式系统。它的搭建涉及多个进程。

### 搭建 Flink 所需组件

Flink 的搭建需要四个不同组件，他们相互协作。这些组件是： JobManager、ResourceManager、TaskManager 和 Dispatcher。所有组件都基于JVM虚拟机
运行。职责如下：

* **JobManager** 作为主进程（master process），JobManager 控制着单个应用程序的执行。换句话说，每个应用都由不同的 JobManager 掌控。JobManager
接收需要执行的应用，该应用包含一个 JobGraph，即逻辑 Dataflow 图，以及打包了全部类、库等资源的JAR文件。JobManager 将 JobGraph 转化成 
ExecutionGraph 的物理 Dataflow 图，图中包含了并行执行的任务。JobManager 向 ResourceManager 申请资源（TaskManager 处理槽）。然后将任务
分发给 TaskManager 执行。执行过程中，其他组件协调，如创建检查点等。
* **ResourceManager** 针对不同的环境和资源提供者（如YARN、Mesos、Kubernetes等），Flink提供了不同的 ResourceManager。ResourceManager
负责管理 Flink 的处理资源单元 TaskManager 处理槽。JobManager 向 ResourceManager 申请处理槽时，ResourceManager 会指示一个拥有空闲处理
槽的 TaskManager，如果无法满足，则 ResourceManager 再和资源提供者通信，让他们提供额外的容器启动 TaskManager。同时 ResourceManager 还负责
终止空闲的 TaskManager 释放资源。
* **TaskManager** 是 Flink 的工作进程（worker process）。通常会启动多个 TaskManager。每个 TaskManager 提供一定数量的处理槽。这个数量就
限制了它能执行的任务数。任务执行期间，运行同一应用的不同任务的 TaskManager 会有数据交换。
* **Dispatcher** 会跨多个作业运行，它提供了REST 接口来让我们提交应用。一旦应用被提交， Dispatcher 会启动一个 JobManager 并将应用转交给它。
Dispatcher 还启动了一个 Web UI，用于提供有关作业执行的信息。

图3-1 展示了应用提交和运行的过程。

![Flink job submit&execute](../../image/bigData/基于Apache%20Flink的流处理/Flink%20job%20submit&execute.png)
**图 3-1：应用提交和组件交互**
 
### 应用部署

TODO
框架模式  
&nbsp; &nbsp; &nbsp; &nbsp;Flink应用打成一个 JAR 文件，通过客户端提交到运行的服务（Dispatcher，Flink JobManager 或者是 YARN
的 ResourceManager）上。

库模式  
&nbsp; &nbsp; &nbsp; &nbsp;在该模式下，Flink 应用会绑定到一个特定应用的容器镜像（如 Docker 镜像中）。镜像中包含运行 JobManager 和 ResourceManager
的代码。Kubernetes 负责启动镜像并确保故障重启。

### 任务执行

一个 TaskManager 允许同时执行多个任务。左侧的 JobGraph 包含了5个算子，其中算子 A 和 C 是数据源，算子 E 是数据汇。字母的下角标是并行度。由于
算子的最大并行度是4，因此至少需要4个处理槽运行。以切片形式调度到处理槽有个好处就是 TaskManager 中的多个任务可以在同一进程执行数据交换而无须访问网络。

![Operator&Task&Slot](../../image/bigData/基于Apache%20Flink的流处理/Operator&Task&Slot.png)
**图 3-2：算子、任务以及处理槽**

TaskManager 会在同一个 JVM 进程内以多线程的方式执行任务。与独立进程相比，线程更加轻量并且通信开销更低，但是任务无法隔离，如果有一个运行异常，可能
会杀死整个 TaskManager 进程。

### 高可用性设置

流式应用通常都会设计成7x24小时运行，因此即便是内部进程发生故障也不能终止运行。

#### TaskManager 故障

部分 TaskManager 发生故障后，JobManager 会向 ResourceManager 申请更多的处理槽。

#### JobManager 故障

JobManager 的故障更加麻烦。它用于控制流式应用执行和保存该过程的元数据。Flink 的高可用基于 Apache ZooKeeper 来完成。

## Flink 中的数据传输

运行过程中会有数据传输。TaskManager 负责数据传输，它会先将数据收集到缓冲区中。换言之，记录不是逐个发送的，而是在缓冲区中以批次形式发送。这是实现
高吞吐的基础。它的机制类似于网络以及磁盘I/O协议中的缓冲技术。

每个 TaskManager 都有一个用于收发数据的网络缓冲池（默认32KB）。流式应用以流水线的方式交换数据，因此每个 TaskManager 之间都要维护一个或多个永久
的TCP 连接来执行数据交换。

当发送任务和接收任务出于同一个 TaskManager 进程时，发送任务会将要发哦是哪个的记录序列化到一个字节缓冲区中，一旦该缓冲区占满就会被放到一个队列里。
接收任务从这个队列里获取缓冲区并将其中的记录反序列化。这意味着同一个 TaskManager 的任务之间没有网络通信。

### 基于信用值的流量控制

Flink 实现了一个基于**信用值**的流量控制机制来降低通信开销，原理如下：接收任务会给发送任务授予一定的信用值。一旦发送端收到信用通知，就会在信用值限定
范围内传输缓冲数据，并附带一定的积压量。

信用值其实就是告诉发送端我有足够资源可以立即传输数据。这个机制还能在出现数据倾斜时有效分配网络资源。

### 任务链接

**任务链接**用于降低本地通信开销。前提：算子有相同的并行度并且通管局哦本地转发通道相连。这个时候多个任务就可以被合并成一个任务，这样就减少了序列化和
通信开销。

但是有时候可能不希望用到它，比如某个任务计算量很大需要对它进行切分。

## 事件时间处理

### 时间戳

在事件时间模式下，Flink 流式应用处理的记录需要包含时间戳。时间戳需要保证随着数流的前进大致递增。

Flink 内部采用8字节的Long值对时间戳进行编码，并将它们以元数据（metadata）的形式附加在记录上。

### 水位线

基于事件时间的应用必须提供水位线（watermark）。水位线用于在事件时间应用中推断每个任务当前的事件时间，时间窗口基于这个时间判断结束边界。

水位线两个基本属性：

1. 必须单调递增。
2. 和记录的时间戳存在联系。

对于延迟记录，Flink 也提供了不同的处理机制。

### 水位线传播和事件时间

任务内部的时间服务（time service）会维护一些计时器（timer），它们依靠接收到水位线来激活。窗口算子为每个活动窗口注册一个计时器，它们会在事件时间
超过窗口的结束时间时清理窗口状态。

当任务接收到水位线时会执行如下操作：

1. 基于水位线记录的时间戳更新内部事件时间时钟。
2. 任务的时间服务找出所有触发时间小于更新后事件时间的计时器。
3. 任务根据更新后的事件时间将水位线发出。

每个分区都维护了一个分区水位线（partition watermark）。当收到某个分区传来的水位线后，任务会以接收值和当前值中较大的那个去更新对应分区水位线的值。
随后，任务会把事件时间时钟调整为所有分区水位线中最小的那个值。

图 3-9 展示了一个有4个输入分区和3个输出分区的任务在接收到水位线后，是如何更新它的分区水位线和时间时间时钟，并将水位线发出。

![水位线传播](../../image/bigData/基于Apache%20Flink的流处理/水位线传播.png)
**图 3-9：利用水位线更新任务的事件时间**

Flink 的水位线处理和传播算法保证了算子任务所发出的记录时间戳和水位线一定会对齐。如果有一个分区的水位线没有前进，或分区空闲下来没有数据，任务的事件
时间时钟就不会前进，继而导致计时器无法触发。

当算子两个输入流的水位线差距很大时，也会产生类似影响。对于两个输入流的任务而言，其事件时间时钟会受制于那个相对较慢的流，较快的流的记录或中间结果会在
状态中缓冲。

### 时间戳分配和水位线生成

时间戳和水位线通常都是在数据流刚进入流处理应用的时候分配和生成的。水位线依赖于时间戳和数据流本身，所以必须显式分配时间戳和生成水位线。Flink DataStream
应用通过三种方式完成该工作：

1. 在数据源完成：可以利用 SourceFunction 在应用读入数据流的时候分配时间戳和生成水位线。如果源函数不再（临时性的）发出水位线，可以把自己声明为空闲。
Flink 会把来自空闲源函数的流分区排除在外。这种机制可以解决上面提到的水位线不向前推进的问题。
2. 周期分配器（periodic assigner）：DataStream API 提供了用户自定义函数 AssignerWithPeriodicWatermarks，它可以从每天记录提取时间戳，
并周期性地响应获取当前水位线的查询请求。
3. 定点分配器（punctuated assigner）：另一个自定义函数是 AssignerWithPunctuatedWatermarks。它可用于需要根据特殊输入记录生成水位线的情况。

用户自定义的时间戳分配函数通常韩进数据源算子，因为在算子处理后，记录顺序和他们的时间戳更难推断。

## 状态管理

大部分的流式应用都有状态。很多算子会不断地读取并更新某些状态，例如：窗口内收集的记录，输入源的读取位置或是一些定制的。可以把状态想象成任务的业务逻辑
所需访问的本地或实例变量。

![Flink job state](../../image/bigData/基于Apache%20Flink的流处理/Flink%20job%20state.png)
**图 3-10：带有状态的流处理任务**

为了让 Flink 的运行层知道算子有哪些状态，算子需要自己对其进行注册。根据作用域不同，状态分为两类：**算子状态**（operator state）和**键值分区状态**
（keyed state）。

### 算子状态

算子状态的作用域是某个算子任务，同一个任务的记录都能访问到这个状态。

Flink 为算子状态提供了三类原语：

**列表状态**（list state）  
&nbsp; &nbsp; &nbsp; &nbsp;将状态表示为一个条目列表。

**联合列表状态**（union list state）  
&nbsp; &nbsp; &nbsp; &nbsp;也是条目列表，但在做状态恢复时，和普通的列表状态不同。

**广播状态**（broadcast state）  
&nbsp; &nbsp; &nbsp; &nbsp;专门为需要保证算子的每个任务状态都相同的场景而设计。这种相同的特性将有利于检查点保存或算子扩缩容。

### 键值分区状态

TO BE CONTINUED

### 状态后端

### 有状态算子的扩缩容

## 检查点、保存点及状态恢复

### 一致性检查点

### 从一致性检查点中恢复

### Flink 检查点算法

### 检查点对性能的影响


### 保存点