---
title: 初识Rocketmq(二)
date: 2017-01-26 08:54:26
tags: [rocketmq]
categories: [消息队列]
description: 本文介绍Rocketmq消息重试机制
---
# Rocketmq消息重试机制
Rocketmq重试机制从三个方面来实现重试的策略。包括生产者向MQ发送消息失败的重试，消费者从MQ拉取消息失败的重试以及消费者处理消息异常时消息的重试。
![](http://ok9lr2dej.bkt.clouddn.com/Rocketmq2.png)
如图所示
1. 生产者的消息重试策略我们通常可以设置重试的最大次数以及重试的超时时间，也就是消息发送到MQ的超时时间。
2. 消费者获取消息的重试策略Rocketmq自身已经为用户实现了，默认是16次尝试，间隔时间为1s，30s,60s......
3. 消费者处理时的异常通过给MQ返回值来判定，ConsumeConcurrentlyStatus.CONSUME_SUCCESS表示处理成功，ConsumeConcurrentlyStatus.RECONSUME_LATER表示处理失败，如果失败则MQ重新发送该消息，直到该消息处理成功。
