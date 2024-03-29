# 第四章 使用 Saga 管理事务

Saga: 一种消息驱动的本地事务序列。

本章将解释如何使用 Saga 来维护数据一致性。之后分析协调 Saga 的两种不同方式：一种是**协同式**（choreography），Saga 的参与方在没有集中控制器
的情况下交换事件式消息；另一种是**编排式**（orchestration），集中控制器告诉 Saga 参与方要执行的操作。

## 4.1 微服务事务管理

只访问一个数据库的单体应用设计简单。接下来介绍为什么微服务架构下的事务会很复杂。

### 4.1.1 微服务架构对分布式事务的需求

在微服务架构下，一个 createOrder() 操作就会访问多个服务，比如 Consumer Service、Kitchen Service 和 Account Service。每个服务都有独立的数据库。
你需要保证多数据库环境下的数据一致性。

### 4.1.2 分布式事务的挑战

传统方式是采用**分布式事务**。分布式事务标准 XA采用**两阶段提交**（two phase commit，2PC）。

分布式事务也有很多问题，有些 NoSQL 数据库，很多消息代理（RabbitMQ、Kafka等）都不支持分布式事务，另外分布式事务本质是同步进程间通信，这会降低分
布式系统的可用性。

未来解决这个问题，我们可以构建松耦合、异步服务的 Saga。

### 4.1.3 使用 Saga 模式维护数据一致性

Saga 由一连串的本地事务组成。使用一系列的状态来完成整个过程的服务调用，当本地事务完成时，状态更改并发布消息。这样的话 Saga 参与方就会减少耦合。  
但是它也会带来一些挑战，一个是服务间缺少隔离，另一个是发生错误时的回滚更改。

#### Saga 使用补偿事务来回滚所做出的改变

假设一个 Saga 的第 _n+1_ 个事务失败了，必须撤销前 _n_ 个事务的影响。每个事务都需要有补偿事务。假如 事务顺序是 T<sub>1</sub>...T<sub>n</sub>，
则对应的补偿事务执行顺序是 C<sub>n</sub>...C<sub>1</sub>。

## 4.2 Saga 的协调模式

* **协同式**：Saga 的决策和执行分布在每个参与方中，它们通过交换时间的方式来进行沟通。
* **编排式**：决策和执行顺序集中在一个 Saga 编排器类中。

### 4.2.1 协同式 Saga

Saga 参与方订阅彼此的事件并作出相应的响应。实现方式：参与方要校验相关的状态信息，并以此判断自己的状态。

### 4.2.2 编排式 Saga

使用命令／异步响应方式与 Saga 的参与方服务通信。Saga 编排器去决定接下来该做什么，该更新为什么状态。

## 4.3 解决隔离问题

Saga 只满足 ACD 三个属性：

* 原子性
* 一致性
* 持久性

### 4.3.1 缺乏隔离导致的问题

缺乏隔离可能导致以下三种异常：

* **丢失更新**：Saga 没有读取更新，而是直接覆盖了另一个服务的更新
* **脏读**：读取到了另外一个 Saga 尚未提交的更新
* **模糊或不可重复读**：一个 Saga 的不同步骤得到了不同的结果，因为另一个 Saga 服务更新了数据

### 4.3.2 相应的对策

使用 *_PENDING 状态来表示正在更新，这被称为**语义锁定对策**。

## 4.4 Order Service 和 Create Order Sage 的设计

...

