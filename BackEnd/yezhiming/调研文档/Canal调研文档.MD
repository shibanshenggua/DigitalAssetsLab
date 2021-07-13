 

 

# **Canal调研文档**

一、概述

Canal是基于MySQL二进制日志的高性能数据同步系统。Canal在阿里巴巴集团（包括https://www.taobao.com）中被广泛使用，以提供可靠的低延迟增量数据管道。

Canal Server能够解析MySQL Binlog并订阅数据更改，而Canal Client可以实现将更改广播到任何地方，例如数据库和Apache Kafka。

具有以下特点：

1.支持所有平台。

2.支持由Prometheus支持的细粒度系统监视。

3.支持通过不同方式（例如通过GTID）解析和预订MySQL Binlog。

4.支持高性能，实时数据同步。

5.Canal Server和Canal Client均支持由Apache ZooKeeper支持的HA /可伸缩性。

6.Docker支持。

canal，主要用途是基于 MySQL 数据库增量日志解析，提供增量数据订阅和消费

早期阿里巴巴因为杭州和美国双机房部署，存在跨机房同步的业务需求，实现方式主要是基于业务 trigger 获取增量变更。从 2010 年开始，业务逐步尝试数据库日志解析获取增量变更进行同步，由此衍生出了大量的数据库增量订阅和消费业务。

基于日志增量订阅和消费的业务包括

数据库镜像

数据库实时备份

索引构建和实时维护(拆分异构索引、倒排索引等)

业务 cache 刷新

带业务逻辑的增量数据处理

当前的 canal 支持源端 MySQL 版本包括 5.1.x , 5.5.x , 5.6.x , 5.7.x , 8.0.x

二、功能框图

![img](http://47.101.69.157:8111/s/9Ycocjg2dLw6pKF/preview)

 

 

三、对比数据同步方案优缺点

 

![img](http://47.101.69.157:8111/s/cEfzGAQyegTq2JM/preview) 

 

四、问题和解决

1，虽然canal服务端解析binlog后不会把数据持久化，但canal服务端会记录每次客户端消费的位置(客户端每次ack时服务端会记录pos点)。如果数据正在更新时，canal服务端挂掉，客户端也会跟着挂掉，mysql依然在插入数据，而redis则因为客户端的关闭而停止更新，造成mysql和redis的数据不一致。解决办法是，只要重启canal服务端和客户端就可以了，虽然canal服务端因为重启之前解析数据清空，但因为canal服务端记录的是客户端最后一次获取的pos点，canal服务端再从这个pos点开始解析，客户端更新至redis，以达到数据的一致。

2，如果只有一个canal服务端和一个客户端，肯定存在可用性低的问题，一种做法是用程序来监控canal服务端和客户端，如果挂掉，再重启；一种做法是多个canal服务端+zk，将canal服务端的配置文件放在zk，任何一个canal服务端挂掉后，切换到其他canal服务端，读到的配置文件的内容就是一致的（还有记录的消费pos点），保证业务的高可用，客户端可使用相同的做法。

3，不使用消息中间件的情况下，一个binlog的数据量大的时候，可能会容易出问题，解决方法就是加消息中间件增加吞吐量。

4，由于canal是一个增量同步的，之前的数据不一致时可能会出错，解决办法就是先同步Redis和mysql里面的数据，后在加入canal。	

五、总结 

Canal的优势可以完全和业务代码进行解耦，可以实时的同步数据库和Redis中的数据，支持通过不同方式（例如通过GTID）解析和预订MySQL Binlog，还是个开源项目，在github上官方问题解决方案也比较完整。

六、原始资料

CanalGitHUb地址：https://github.com/alibaba/canal

常见问题解决：https://blog.csdn.net/Robin_hc/article/details/90600281

 

 