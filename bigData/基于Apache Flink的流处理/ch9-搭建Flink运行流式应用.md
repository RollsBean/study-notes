# 第九章 - 搭建Flink运行流式应用

## 部署模式

Flink 支持多种环境下的部署，例如本地机器、裸机集群、YARN 集群或 Kubernetes 集群。本节介绍如何配置并启动 Flink。

### 独立集群

独立集群至少包含一个主进程和一个 TaskManager 进程，它们可以运行在一台或多台机器上。

**启动过程：**  
1. 主进程会启动单独的县城来运行 Dispatcher 和 ResourceManager。
2. 运行之后，TaskManager 就会将自己注册到 ResourceManager 中。

![flink-standalone-submit](../../image/bigData/基于Apache%20Flink的流处理/flink-standalone-submit.png)

**客户端提交作业过程**  
1. 客户端将作业提交到 Dispatcher
2. 后者会在内部启动一个 JobManager 线程并提供 JobGraph。
3. JobManager 向 ResourceManager 申请处理槽
4. 执行任务

执行 flink 安装包中的 _./bin/start-cluster.sh_ 脚本，它会在本地启动一个主进程和一个或多个 TaskManager。启动之后请求 _http://localhost:8081_ 
来访问 Flink Web UI。

如果要在多台机器上搭建 Flink 集群：  
* 将 TaskManager 的主机名（或 IP）列在 _./conf/slaves_ 文件中。
* _./bin/start-cluster.sh_ 脚本需要配置所有 TaskManager 主机的 SSH 登录。
* Flink 目录路径需要一致。
* _./conf/flink-conf.yaml_ 文件中将 **jobmanager.rpc.address** 配置为主进程的主机名（或 IP）。

配置完成后使用 _./bin/start-cluster.sh_ 脚本启动，使用 _./bin/stop-cluster.sh_ 停止集群。

### Docker

需要启动两类容器，一个主容器（master container）和多个运行 TaskManager 的容器（worker container）。

```shell
// 启动主进程
docker run -d --name flink-jobmanager \
  -e JOB_MANAGER_RPC_ADDRESS=jobmanager \
  -p 8081:8081 flink:1.7 jobmanager

// 启动工作进程，名字不要重复
docker run -d --name flink-taskmanager-1 \
  --link flink-jobmanager:jobmanager \
  -e JOB_MANAGER_RPC_ADDRESS=jobmanager flink:1.7 taskmanager
```

Docker 镜像包含了主容器和工作容器，只不过启动参数不同。启动之后就可以通过 CLI 客户端（_./bin/flink_）提交应用。

### Apache Hadoop YARN

Flink 以两种模式和 YARN 进行集成：**作业模式**（job mode）和**会话模式**（session mode）。作业模式下，Flink 集群启动后只运行单个作业，作业
结束，集群就会停止。

**作业模式提交作业流程：**  
1. 客户端请求 YARN 的 ResourceManager（RM） 提交作业
2. ResourceManager 会启动一个新的进程，包含一个 JobManager 线程和一个 ResourceManager。
3. JobManager 向 ResourceManager 请求处理槽。
4. Flink 的 ResourceManager 向 YRAN 的 RM 申请容器并启动 TaskManager 进程。
5. 随后，TaskManager 注册在 Flink 的 ResourceManager 中并提供给 JobManager。
6. 最后，JobManager 将作业提交到 TaskManager 运行。

会话模式下，系统会启动一个长时间运行的 Flink 集群，它可以运行多个作业，需要我们手工停止，也就是说不同的作业会运行在一个集群中。

### Kubernetes

常用术语：

* Pod 是由 Kubernetes 管理的容器（比如 Docker）
* 一个 Deployment 定义了一组需要运行的特定数量的 Pod 或容器。Deployment 可以自由伸缩。
* Kubernetes 会在集群任意位置运行 Pod。当 Pod 重启时， 它们的 IP 可能会变化。为此，Kubernetes 还提供了 Service（路由的概念，提供了类似服务
注册发现的功能），它定义了 Pod 的访问策略。

在 Kubernetes 上搭建 Flink 需要两个 Deployment，一个主进程 Pod，另一个 工作进程 Pod。此外，还有一个 Service，用于暴露主进程的 Pod 给工作进程。

Flink Docker 镜像和主进程 Deployment 完全相同，只是启动参数不一样（args: -taskmanager）。

## 高可用性设置

Flink 的 JobManager 中存放了应用以及和它执行有关的元数据。这些信息需要主进程发生故障时进行恢复。Flink 的 HA 依赖于 ZooKeeper 以及持久化存储
（例如 HDFS、S3）。

### 独立集群的 HA 设置

独立集群需要后备 Dispatcher 和 TaskManager 进程，用于接管故障进程的工作。只有保证有足够资源（处理槽）接替故障 TaskManager 工作即可。

如果配置了 HA，所有的 Dispatcher 就会在 ZooKeeper 中注册。

### YARN 上的 HA 设置

Flink 的主进程会作为 YARN 的 ApplicationMaster 启动，YARN 默认会重启发生故障的容器（包括主进程和工作进程）。可以在 _yarn-site.xml_
通过以下配置设置 ApplicationMaster 最大重启次数；

```xml
<property>
    <name>yarn.resourcemanager.am.max-attempts</name>
    <value>4</value>
    <description>
        默认最大次数是 2，即最多重启一次。
        这里设置为 4
    </description>
</property>
```

此外，还需要在 _./conf/flink-conf.yaml_ 配置应用的最大重启次数：

```properties
# 最多重启 3 次，也包括首次启动。
yarn.application-attempts: 4
```

### Kubernetes 的 HA 设置

需要调整 Flink 的配置并提供一些信息，例如在 _./conf/flink-conf.yaml_ 中配置 ZooKeeper Quorum 节点主机名，持久化路径和 Flink 集群 ID

## 集成 Hadoop 组件

Flink 可以轻易地和 Hadoop 生态组件集成。无论什么组件都需要将 Hadoop 依赖加入到 Flink 的 Classpath 中。配置方式有三种：

* 使用构建好的 Flink 发行版。
* 可用于，两者版本不匹配的情况下。例如其他厂商的 Flink。这个方法需要使用 Flink 源码构建，使用以下两个命令之一：  
```shell
／／ 针对特定官方版本 Hadoop
mvn clean install -DskipTests -Dhadoop.version=2.6.1

／／ 针对特定发行商的 Hadoop 构建
```
* 不带 Hadoop 的 Flink 发行版并为 Hadoop 依赖手动配置 Classpath。在可以访问 Hadoop 命令的前提下，可以通过设置：
export HADOOP_CLASSPATH=`hadoop classpath`。

## 文件系统配置

在 Flink 中，每个文件系统都由 org.apache.flink.core.fs.FlieSystem 类的一个实现来表示。

## 系统配置

Flink 提供了很多配置，这些配置可以在 _./conf/flink-conf.yaml_ 文件中定义。

### Java 和类加载

Flink 为每个作业的类注册到一个独立的用户代码类加载器中，这样可以保证作业依赖。

### CPU

Flink 不主动限制 CPU 资源量，但会采用处理槽来控制工作进程的任务数。

### 内存和网络缓冲

通常，主进程对于内存的要求并不高，它默认的 JVM 堆内存数量只有 1GB。除了 JVM，Flink 的网络栈和 RocksDB 也会消耗很多内存。Flink 网络栈基于 Netty，
他会在堆外内存分配网络缓冲区。可以通过 taskmanager.network.memory.fraction 配置分配比例，默认是 JVM 堆内存的 10%。

### 检查点和状态后端

利用 state.backend 参数定义集群的默认状态后端。

### 安全性

Flink 支持 Kerberos 身份验证。