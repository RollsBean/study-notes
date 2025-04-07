# Java性能工具箱

## 3.1 操作系统工具和分析
### 3.1.1 CPU使用率
CPU使用率分为两类：用户时间和系统时间。**用户时间**指的是 CPU 执行应用程序代码所占的百分比，**系统时间**是执行内核代码占的百分比。

另外，CPU 使用率是一段时间的平均值，比如5秒、10秒。

Linux 上运行 `vmstat 1` 得到每秒 CPU 执行用户应用程序的时间、系统内核时间和空闲时间

### 3.1.2 CPU运行队列
### 3.1.3 磁盘使用率
执行 `iostat -xm 5` 查看内存统计数据，打印的信息包括：io wait 占比、空闲占比、磁盘使用率、每秒写入、读出的数据量等指标
### 3.1.4 网络使用率
网络监控工具 netstat，一般使用 nicstat 工具

## 3.2 Java监控工具
JDK 自带的工具如下：

* jcmd： 
  打印 Java 进行中的基本类、线程和 JVM 信息
* jconsole:
  图形化视图
* jmap:
  提供堆转储和其他 JVM 内存使用的信息。
* jinfo:
  查看JVM系统属性
* jstack:
  提供 GC 和类加载活动的信息

在 Docker 中通过 `docker exec` 命令运行。

### 3.2.1 基本的VM信息
JVM 运行时间：
```shell
jcmd <pid> VM.update
```

系统属性：
```shell
jcmd <pid> VM.system_properties
```

JVM 版本：
```shell
jinfo -sysprops <pid>
```

JVM 调优标志
```shell
jcmd <pid> VM.flags [-all]
```

## 3.3 性能分析工具
分析工具一般也是 Java 写的。
### 3.3.1 采样分析器
性能分析有两种模式：采样模式和探查模式。

可以使用**火焰图**（flame graph）显示调用栈。火焰图是自底向上的图表，展示了占用 CPU 最多的方法。
### 3.3.2 探查分析器
相对于采样分析器更有侵入性，它在类加载时更改字节码序列。
### 3.3.3 阻塞方法和线程时间线
jvisualvm 内置了性能分析器，可以显示 CPU 执行时间、空闲时间等。
### 3.3.4 原生分析器
类似 async-profiler 和 Oracle Developer Studio 的工具，除了可以分析 Java 代码，还可以分析原生代码。