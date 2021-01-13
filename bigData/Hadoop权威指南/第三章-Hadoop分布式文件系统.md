# 第三章 Hadoop分布式文件系统
<br>

当数据集超过一台物理计算机的储存能力时，就要对其进行分区。管理网络中跨多台计算机存储的文件系统称为**分布式文件系统**。

## HDFS的设计

HDFS 以流式数据访问模式来存储超大文件。

**超大文件**

> "超大文件"指至少具有几百MB、几百GB的文件

**流式数据访问**

> 一次写入，多次读取是最高效的访问模式，数据集通常由数据源生成并长时间在数据集上分析，所以读取整个数据集延迟比读取第一条记录延迟要更重要

**商用硬件**

> 普通的商用硬件作为集群

**低时间延迟的数据访问**

> 低延迟的数据访问不适合在hdfs上运行，例如几十毫秒的延迟。hdfs为数据高吞吐量优化，代价就是高时间延迟

**大量的小文件**

namenode将文件系统的元数据存储在内存中，因此文件系统所能存储的文件数受限于namenode的内存容量。

**多用户写入，任意修改文件**

文件可能只有一个 writer，写操作总是将数据添加到文件末尾，它不支持多个写入操作，也不支持任意位置修改（目前是否支持要再查阅资料）

## HDFS 的概念

### 数据块

磁盘：单个磁盘文件系统+多个磁盘块
HDFS 块（block）默认64MB。文件被划分为多个分块（chunk）

**为何这么大**

为了最小化寻址开销，这样一次寻址可以传输一整个块（64MB），数据传输远大于寻址的时间。

### namenode 和 datanode

HDFS以管理者-工作中模式运行，即一个namenode(管理者) 和多个datanode(工作者)。<br/>
namenode 管理文件系统的命名空间，它维护着文件系统树及树内的所有文件和目录。这些信息永久存储在本地磁盘上：**命名空间镜像文件**
和**编辑日志文件**。<br/>

**客户端** 通过namenode和datanode交互来访问整个文件系统。工作节点datanode受namenode和客户端调度，并定期向namenode返送它们所存储的块的列表。
没有namenode，文件系统无法使用。如果namenode损坏，文件系统上的文件将丢失，因为无法根据datanode的块重建文件。

## 命令行接口

### 基本文件操作

`fs -help`获取详细帮助文件

从本地复制一个文件到HDFS
```shell script
hadoop fs -copyFromLocal input/docs/a.txt hdfs://localhost/user/a.txt
```
hdfs://localhost 可以省略，因为已经在*core-site.xml*中指定,所以hdfs路径可以写成 */user/a.txt*(绝对路径)或者*a.txt*
复制到hdfs home目录中<br/>

从HDFS复制文件到本地
```shell script
hadoop fs -copyFromLocal a.txt a.copy.txt
```

创建文件夹和查看文件列表：
```shell script
hadoop fs mkdir books
hadoop fs -ls .
```
查看文件返回的结果与Unix ls -l 类似。

## Hadoop 文件系统

HDFS是Hadoop抽象文件系统的一个实现。 Java抽象类 `FileSystem`定义了文件系统接口，

### 接口

Hadoop是Java写的，Java API 可以调用所有的Hadoop文件系统。文件解释器也是一个Java应用。

#### Thrift

Thrift API 通过把Hadoop文件系统包装成 Apache Thrift服务来访问。需要使用时，只需运行Thrift服务，并以代理方式访问Hadoop文件系统

#### HTTP

只读接口。namenode中web服务器（50070端口）以xml提供目录列表，datanode的web服务器（50075端口）提供文件数据传输。

#### FTP

使用FTP协议与HDFS交互。









