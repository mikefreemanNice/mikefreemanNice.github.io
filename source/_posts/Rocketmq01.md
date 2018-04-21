---
title: Rocketmq多master搭建
date: 2017-01-24 10:57:07
tags: [rocketmq]
categories: [消息队列]
description: 本文介绍如何搭建rocketmq的多master的环境。
---
# Rocketmq物理部署结构图
![](http://ok9lr2dej.bkt.clouddn.com/rocketmq1.png)
# 安装包下载
** 注意： 以下操作每台机器都需要配置 **
Rocketmq目前已收录于apache下，目前版本是4.0。在此之前版本更新到3.5.8，github上release版本提供了3.5.8的源码，稳定的编译后的版本是3.2.6，所以这里我以3.2.6介绍。rocketmq3.x下载[点这里](https://github.com/alibaba/RocketMQ/releases)。
有兴趣的童鞋可以尝试4.x，地址在[这里](https://rocketmq.incubator.apache.org/docs/quick-start/)，源码采用maven编译即可，这里不过多赘述。
# 服务器环境
| IP| 用户名| 角色| 模式|
|---|---|---|---|
|192.168.2.201|root|rocketmq-nameserver1,rocketmq-master1|Master1|
|192.168.2.202|root|rocketmq-nameserver2,rocketmq-master2|Master2|
# Hosts配置信息
| IP| Name|
|---|---|
|192.168.2.201|rocketmq-nameserver1|
|192.168.2.201|rocketmq-master1|
|192.168.2.202|rocketmq-nameserver2|
|192.168.2.202|rocketmq-master2|
|192.168.2.201|centos201|
|192.168.2.202|centos202|
这里需要注意一点，我在hosts信息中分别把各自主机名和对应ip配置进来，之前尝试没有配置而报错。

# 上传解压
- 上传alibaba-rocketmq-3.2.6.tar.gz文件到/usr/local/software
- 解压并创建软链接
```
tar -zxvf alibaba-rocketmq-3.2.6.tar.gz -C /usr/local
ln -s alibaba-rocketmq rocketmq
```

# 创建存储路径
```
mkdir /usr/local/rocketmq/store
mkdir /usr/local/rocketmq/store/commitlog
mkdir /usr/local/rocketmq/store/consumequeue
mkdir /usr/local/rocketmq/store/index
```

# 修改Rocketmq的配置文件
```
vim /usr/local/rocketmq/conf/2m-noslave/broker-a.properties
vim /usr/local/rocketmq/conf/2m-noslave/broker-b.properties
```
***

```
brokerClusterName=rocketmq-cluster
#broker名字，注意此处不同的配置文件填写的不一样
brokerName=broker-a
#0 表示 Master，>0 表示 Slave
brokerId=0
#nameServer地址，分号分割lushDiskType=ASYNC_FLUSH
namesrvAddr=rocketmq-nameserver1:9876;rocketmq-nameserver2:9876
#在发送消息时，自动创建服务器不存在的topic，默认创建的队列数
defaultTopicQueueNums=4
#是否允许 Broker 自动创建Topic，建议线下开启，线上关闭
autoCreateTopicEnable=true
#是否允许 Broker 自动创建订阅组，建议线下开启，线上关闭
autoCreateSubscriptionGroup=true
#Broker 对外服务的监听端口
listenPort=10911
#删除文件时间点，默认凌晨 4点
deleteWhen=04
#文件保留时间，默认 48 小时
fileReservedTime=120
#commitLog每个文件的大小默认1G
mapedFileSizeCommitLog=1073741824
#ConsumeQueue每个文件默认存30W条，根据业务情况调整
mapedFileSizeConsumeQueue=300000
#destroyMapedFileIntervalForcibly=120000
#redeleteHangedFileInterval=120000
#检测物理文件磁盘空间
diskMaxUsedSpaceRatio=88
#存储路径
storePathRootDir=/usr/local/rocketmq/store
#commitLog 存储路径
storePathCommitLog=/usr/local/rocketmq/store/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/usr/local/rocketmq/store/consumequeue
#消息索引存储路径
storePathIndex=/usr/local/rocketmq/store/index
#checkpoint 文件存储路径
storeCheckpoint=/usr/local/rocketmq/store/checkpoint
#abort 文件存储路径
abortFile=/usr/local/rocketmq/store/abort
#限制的消息大小
maxMessageSize=65536
#flushCommitLogLeastPages=4
#flushConsumeQueueLeastPages=2
#flushCommitLogThoroughInterval=10000
#flushConsumeQueueThoroughInterval=60000
#Broker 的角色
#ASYNC_MASTER 异步复制Master
#SYNC_MASTER 同步双写Master
#SLAVE
brokerRole=ASYNC_MASTER
#刷盘方式
#ASYNC_FLUSH 异步刷盘
#SYNC_FLUSH 同步刷盘
flushDiskType=ASYNC_FLUSH
#checkTransactionMessageEnable=false
#发消息线程池数量
#sendMessageThreadPoolNums=128
#拉消息线程池数量
#pullMessageThreadPoolNums=128
```

# 修改日志配置文件
```
mkdir -p /usr/local/rocketmq/logs
cd /usr/local/rocketmq/conf && sed -i 's#${user.home}#/usr/local/rocketmq#g' *.xml
```

# 修改启动脚本参数
开发环境jvm配置
```
vim /usr/local/rocketmq/bin/runbroker.sh
```
```
JAVA_OPT="${JAVA_OPT}" -server -Xms1g -Xmx1g -Xmn512m
-XX:PermSize=128m -XX:MaxPermSize=320m"
```
***
```
vim /usr/local/rocketmq/bin/runserver.sh
```
```
JAVA_OPT="${JAVA_OPT}" -server -Xms1g -Xmx1g -Xmn512m
-XX:PermSize=128m -XX:MaxPermSize=320m"
```

# 启动NameServer
```
cd /usr/local/rocketmq/bin
nohup sh mqnamesrv &
```

# 启动BrokerServer
- brokerA(192.168.2.201)
```
cd /usr/local/rocketmq/bin
nohup sh mqbroker -c /usr/local/rocketmq/conf/2m-noslave/broker-a.properties >/dev/null 2>&1 &
```
- brokerB(192.168.2.202)
```
cd /usr/local/rocketmq/bin
nohup sh mqbroker -c /usr/local/rocketmq/conf/2m-noslave/broker-b.properties >/dev/null 2>&1 &
```
查看是否启动成功
```
netstat -ntlp
jps
tail -f -n 500 /usr/local/rocketmq/logs/rocketmqlogs/broker.log
tail -f -n 500 /usr/local/rocketmq/logs/rocketmqlogs/namesrc.log
```

# 停止服务及数据清理
```
cd /usr/local/rocketmq/bin
sh mqshutdown broker
sh mqshutdown namesrv
```
```
rm -rf /usr/local/rocketmq/store
mkdir /usr/local/rocketmq/store
mkdir /usr/local/rocketmq/store/commitlog
mkdir /usr/local/rocketmq/store/consumequeue
mkdir /usr/local/rocketmq/store/index
```

# 多Master多Slave模式
配置多master多slave模式的方式和本文介绍的多master方式大体相同，不过需要注意以下几点。
- Hosts配置信息

| IP| Name|
|---|---|
|192.168.2.201|rocketmq-nameserver1|
|192.168.2.201|rocketmq-master1|
|192.168.2.202|rocketmq-nameserver2|
|192.168.2.202|rocketmq-master2|
|192.168.2.203|rocketmq-nameserver3|
|192.168.2.203|rocketmq-master1-slave|
|192.168.2.204|rocketmq-nameserver4|
|192.168.2.204|rocketmq-master2-slave|
|192.168.2.201|centos201|
|192.168.2.202|centos202|
|192.168.2.203|centos203|
|192.168.2.204|centos204|

- 选择配置文件
这里需要编辑对应的配置文件：在/usr/local/rocketmq/conf下存在三个配置文件，分别为2m-2s-async,2m-2s-sync,2m-noslave。当我们需要搭建多主多从的集群环境时，考虑异步复制还是同步双写模式来选择对应配置文件进行修改。
- 配置文件修改
配置文件修改注意以下几点
```
brokerClusterName=rocketmq-cluster
#broker名字，注意此处不同的配置文件填写的不一样,若为从节点，则与对应主节点名字相同
brokerName=broker-a
#0 表示 Master，>0 表示 Slave
brokerId=0
#nameServer地址，分号分割lushDiskType=ASYNC_FLUSH
namesrvAddr=rocketmq-nameserver1:9876;rocketmq-nameserver2:9876;rocketmq-nameserver3:9876;rocketmq-nameserver4:9876
#Broker 的角色
#ASYNC_MASTER 异步复制Master
#SYNC_MASTER 同步双写Master
#SLAVE
brokerRole=ASYNC_MASTER
```
