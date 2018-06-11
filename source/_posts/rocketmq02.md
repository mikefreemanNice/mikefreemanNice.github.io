---
title: 初识Rocketmq(一)
date: 2017-01-20 13:21:53
tags: [rocketmq]
categories: [消息队列]
comments: false
description: 本文介绍Rocketmq产品历史和几个专业术语以及集群方式
---

# 产品发展历史
- Metaq（Metamorphosis） 1.x

  由开源社区 killme2008 维护，开源社区非常常活跃。

  [https://github.com/killme2008/Metamorphosis](https://github.com/killme2008/Metamorphosis)
<!--more-->
- Metaq 2.x

  亍2012年10月份上线，在淘宝内部被广泛使用。

- RocketMQ 3.x

  Metaq正式更名为Rocketmq，集群由第三方的zookeeper转化为自身的nameserver。开源社区地址：[https://github.com/alibaba/RocketMQ](https://github.com/alibaba/RocketMQ)
- RocketMQ 4.x
  Rocketmq捐赠于apache，开源地址：[https://github.com/apache/incubator-rocketmq](https://github.com/apache/incubator-rocketmq)

# 专业术语
- **Producer**：
消息生产者，负责产生消息，一般有业务系统产生消息。
- **Consumer**：
消息消费者，负责消费消息，一般由后台系统进行异步消费。
- **PushConsumer**：
Consumer的一种，应用通常向Consumer对象注册一个listener接口，一旦接受消息，Consumer对象立刻回调listener接口方法。
- **PullConsumer**：
Consumer的一种，应用通常主动调用Consumer的拉消息方法从Broker拉消息，主动权由应用控制。
- **Producer Group**：
一类Producer的集合名称，这类Producer通常収送一类消息，且发送逻辑一致。
- **Consumer Group**：
一类Consumer的集合名称，这类Consumer通常消费一类消息，且发送逻辑一致。
- **Broker**：
消息中转角色，负责存储消息，转发消息，一般也称为 Server。在 JMS 规范中称为 Provider。
- **广播消息**：
一条消息被多个Consumer消费，即使这些Consumer属于同一个Consumer Group，消息也会被Consumer Group的每个Consumer都消费一次，广播消费中的Consumer Group概念可以认为在消息划分方面没有意义。在CORBA Notification规范中，消费方式都属于广播消费。在JMS规范中，相当于JMS publish/subscribe model。
- **集群消费**：
一个 Consumer Group 中的 Consumer 实例平均分摊消费消息。例如某个 Topic 有 9 条消息，其中一个Consumer Group 有 3 个实例（可能是 3 个进程程，或者 3 台机器），那么每个实例只消费其中的 3 条消息。在 CORBA Notification 规范中，无此消费方式。在 JMS 规范中， JMS point-to-point model 与之类似，但是 RocketMQ 的集群消费功能大等亍 PTP 模型。因为 RocketMQ 单个 Consumer Group 内的消费者类似亍 PTP ，但是一个 Topic/Queue 可以被多个 Consumer Group 消费。
- **顺序消息**：
消费消息的顺序要同发送消息的顺序一致，在Rocketmq中，主要指的是局部顺序，即一类消息为满足顺序性，Producer必须单线程顺序发送，且发送到同一个队列，这样Consumer就可以按照Producer发送的顺序进行消费。
- **普通顺序消息**：
顺序消息的一种。正常情况下可以保证完全的顺序消息，但是一旦通信发生异常，Broker重启，由于队列总数发生变化，哈希取模后定位的队列发生变化，产生短暂的消息顺序不一致，如果业务可以容忍这种情况，则普通顺序消息比较合适。
- **严格顺序消息**：
顺序消息的一种，无论正常或异常情况都可以保证顺序消息，但是牺牲了分布式failover特性，即Broker集群中只有有一台机器不可用，则整个集群都不可用，服务可用性大大降低。目前已知的应用只有数据库的binlog同步强依赖严格顺序消息，其他应用绝大部分都可以容忍短暂的乱序，推荐使用普通的顺序消息。
- **Message Queue**：
在Rocketmq中，所有消息队列都是持久化，长度无限的数据结构，所谓长度无限是指队列中的每个存储单元都是定长，访问其中的存储单元使用Offset访问，offset为java lang类型，64位，理论上讲100年内不会益出，所以认为是长度无限，另外队列只保存最近几天的数据，之前的数据会按照过期时间来删除。也可以认为Message Queue是一个长度无限的数组，offset是下标。

# 集群方式
- **单个Master**
这种方式的风险比较大，一旦Broker重启或者宕机，会导致整个服务不可用，不建议线上环境使用。
- **多个Master模式**
一个集群中午slave，都是master，例如两个master或者3个master。
优点：单个master宕机或者重启维护对应用无影响，在磁盘配置为raid10时，即使机器宕机不可恢复情况下，消息也不会丢失（异步刷盘丢失少量数据，同步刷盘消息不丢失），性能最高。
缺点：单台机器宕机期间，这台机器未被消费的消息在机器恢复之前不可订阅，消息实时性会受到影响。
启动方式：先启动nameserver，再启动master1，master2...
- **多Master，多Slave模式，异步复制**
每个master配置一个或多个slave，有多对master-slave，HA采用异步复制方式，主备有短暂消息延迟，毫秒级。
优点：即使磁盘损坏，消息丢失的非常少，且消息的实时性不会受影响，因为Master宕机后，消息仍然可以从Slave消费，此过程对应用透明，不需要人工干预，性能同多master模式几乎一样。
缺点：Master宕机，磁盘损坏情况下，会丢失少量数据。
启动方式：先启动nameserver，再启动第一个master，第二个master，第一个slave，第二个slave。
- **多Master，多Slave模式，同步双写**
每个Master配置一个或多个slave，有多对master-slave，HA采用同步双写方式，主备都写成功，向应用返回成功。
优点：数据与服务都无单点，Master宕机情况下，消息无延迟，服务可用性与数据可用性都非高。
缺点：性能比异步复制模式略低，大约低10%左右，发送单个消息的RT会略高。
启动方式：先启动nameserver，再启动第一个master，第二个master，第一个slave，第二个slave。
