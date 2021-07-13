# **Pulsar调研报告**

#### 一、**概念**

Pulsar 是一个用于服务器到服务器的消息系统，具有多租户、高性能等优势。 Pulsar 最初由Yahoo 开发，目前由Apache 软件基金会管理。

#### 二、**关键特征**

Pulsar 的关键特性如下：

1、Pulsar 的单个实例原生支持多个集群，可跨机房在集群间无缝地完成消息复制。

2、极低的发布延迟和端到端延迟。

3、可无缝扩展到超过一百万个 topic。

4、简单的客户端 API，支持 Java、Go、Python 和 C++。

5、支持多种topic 订阅模式独占订阅、共享订阅、故障转移订阅）

6、通过 Apache BookKeeper提供的持久化消息存储机制保证消息传递。

①　由轻量级的 serverless 计算框架 Pulsar Functions实现流原生的数据处理。

②　基于Pulsar Functions的Serverless Connector框架 Pulsar IO使得数据更易移入、移出Apache Pulsar。

③　分层式存储可在数据陈旧时，将数据从热存储卸载到冷/长期存储(如S3、GCS)中

 

#### 三、**架构**

![img](http://47.101.69.157:8111/s/AjPZ6QrApoX2i6t/preview)

#### 四、**与其他消息中间件对比**

|               | Pulsar                               | Kafka                                                        | RabbitMQ                       | RocktMQ                     |
| ------------- | ------------------------------------ | ------------------------------------------------------------ | ------------------------------ | --------------------------- |
| 安装所需环境  | 1. java环境2. Pulsar安装包包含其主键 | 1. Java环境2. Zookeeper环境（kafka2.8可以不依赖）3. Kafka安装包 | 1. Erlang环境2. RabbitMQ安装包 | 1. JAVA环境2. RocktMQ安装包 |
| 包含组件      | Zookeeper、Bookkeeper、Broker        | Broker、Zookeeper                                            | RabbitMQ-Server                | NameServer、Broker          |
| 峰值吞吐量    | 305MB/S                              | 605MB/S                                                      | 38MB/S                         |                             |
| 机器消耗      | 最少3,官方推荐是6,最多9              | 最少3                                                        | 3                              | 最少3，最多6                |
| P99延迟(毫秒) | 5毫秒(200MB/S负载)                   | 25毫秒(200MB/S负载)                                          | 1ms*（减少至30MB/s的负载）     |                             |

 

***\*功能维度：\****

RabbitMQ（3.8.4）、RocketMQ（4.7.1）、Pulsar（2.6.1）这三款产品大都支持这些常见功能，值得注意的是：
RabbitMQ不支持延迟队列（定时消息）、消息过滤、消息回溯、消息保留。
RocketMQ不支持优先级队列、非持久化主题、消息生存时间、多租户、多中心。
Pulsar不支持事务消息（2020年10月底预计发布2.7.0版本包含预览版的事务消息功能）、优先级队列、消息过滤。

***\*高可用维度：\****

***\*RabbitMQ高可用\****：

RabbitMQ 只有Rabbitmq-server这一个组件，依赖「镜像队列」功能实现镜像集群：每个RabbitMQ节点既保存有队列相同的元数据，又保存有队列实际的消息数据。 任一节点宕机，不影响消息在其他节点上进行消费。缺点在于：

1、性能开销非常大，因为要同步消息到对应的节点，这个会造成网络之间的数据量的频繁交互，对于网络带宽的消耗和压力都是比较重的

2、扩展性差，rabbitMQ是集群，但不是分布式的，所以当某个Queue负载过重，并不能通过新增节点来缓解压力，因为所有节点上的数据都是相同的，这样就没办法进行扩展了

3、镜像队列不是负载均衡，因为每个操作在所有节点都要做一遍。镜像队列无法提升消息的传输效率，或者更进一步说，由于镜像队列会在不同节点之间进行同步，会消耗消息的传输效率。

 

  ***\*RocketMQ高可用：\****

RocketMQ集群包含两个组件：NameServer集群、Broker集群。每台NameServer 机器都拥有所有的路由信息，包括所有的 Broker 节点信息、数据信息等 ，这样只要有一台 NameServer 存活就不会影响系统的稳定性。若Master Broker 挂掉，RocketMQ4.5 版本之前，是人工运维，通过人手工切换Master Broker，RocketMQ4.5 之后，通过Dledger 技术以及Raft 协议进行leader 选主。整个过程很快，大概十几秒或者几十秒就能完成切换动作，完全的全自动的将Slave Broker 选为Master broker 对外提供服务，实现高可用模式。

***\*Pulsar高可用：\****

Pulsar 集群包含 3 个组件：ZooKeeper 集群、BookKeeper 集群和 Broker 集群。

ZooKeeper 集群通常由一组机器组成，一般 3 台以上就可以组成一个可用的 ZooKeeper 集群。

组成 ZooKeeper 集群的每台机器都会在内存中维护当前的服务器状态，并且每台机器之间都会互相保持通信。只要集群中存在超过一半的机器能够正常工作，那么整个集群就能够正常对外服务。

Broker 集群中的各个broker 是无状态的，如果拥有某个Topic的Broker崩溃，则将该Topic立即重新分配给另一个Broker。当发生 Topic 的迁移时，Pulsar 只是将所有权从一个 Broker 转移到另一个 Broker，在这个过程中，不会有任何数据复制发生。

Broker 的无状态性质使动态分配成为可能，因此可以根据使用情况快速扩展或收缩集群。

BookKeeper 集群包含一组Bookies节点，一个Topic由多个Ledger构成，一个Ledger由一个或多个Fragment组成，每个Fragment有多个条目Entry组成，每个Entry上包含的就是消息Message。Fragments分布在Bookie集群中，跨多个Bookies带状分布。存储可以单独扩展。如果存储是瓶颈，那么只需要添加更多的Bookies，他们会自动承担负载，不需要Rebalance。当Bookie不可用时，自动恢复模式将自动进行数据重新复制到其他的Bookies。

此外，BookKeeper 通过 Quorum Vote 的方式来实现数据的一致性，跟 Master/Slave 模式不同，BookKeeper 中每个节点也是对等的，对一份数据会并发地同时写入指定数目的存储节点。对等的存储节点，保证了多个备份可以被并发访问；也保证了存储中即使只有一份数据可用，也可以对外提供服务。

#### 五、**总结**

从GitHub上各社区的数据对比，官方仓库start数目上，RocketMQ基本上是另外两款产品的两倍；Issues数目上来看，Pulsar的open数量是RabbitMQ的将近11倍，是RocketMQ的5倍，说明Pulsar还在一个快速迭代的过程中，相对于另外两款欠缺一些稳定性。

 

#### 六、**原始资料**

**Pulsar官方文档：**[**http://pulsar.apache.org/docs/zh-CN/next/standalone/**](http://pulsar.apache.org/docs/zh-CN/next/standalone/)

**Pulsa机器消耗对比:https://zhuanlan.zhihu.com/p/265423678**

 