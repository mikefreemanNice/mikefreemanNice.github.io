---
title: SpringCloud环境搭建
date: 2018-10-02 15:44:53
tags: [SpringCloud]
comments: false
categories: [SpringCloud]
description: 如何用SpringCloud搭建微服务
---

# 问题引入
这篇文章的目的是让读者了解如何通过SpringCloud来搭建服务，与Dubbo，Thrift这两款组件作为微服务中间件需要有专门的团队进行扩展维护。SpringCloud作为很火的开源项目，提供了很全面的微服务组件。本篇文章不会详细的描述搭建过程，因为网络上这种文章太多，笔者写不出新颖的东西，不过笔者可以把搭建的思路这这里描述一下。

# 术语普及
首先是注册中心Eureka、Consul，这里需要说明一点，Eureka其实指的事Eureka-Server，它和Consul都是类似ZK的集群。  
应用服务即业务服务，比如用户模块、支付模块、订单模块等等。这里就是Eureka-Client，当然这里说的服务是指服务提供方即（Provider）。  
那么服务调用方（Consumer）也相当于Eureka-Client，它调用服务的时候不像Dubbo看成一个Bean，而是通过http服务方式访问，Spring-Cloud提供了一个组件Fegin，可以进行调用。 
 
- EurekaServer == Consul == Zookeeper (Cluster)  
- EurekaClient == Provider (Cluster)  
- EurekaClient == Consumer (Cluster)


# 帮助文档
这里推荐一个比较全的简单流程：[https://github.com/dyc87112/SpringCloud-Learning](https://github.com/dyc87112/SpringCloud-Learning)  
当然看Spring-Cloud的官网文档比较好： [https://spring.io/guides](https://spring.io/guides)  
Sping-Cloud中文文档： [https://springcloud.cc/spring-cloud-dalston.html](https://springcloud.cc/spring-cloud-dalston.html)

# 注意事项
SpringCloud的最大问题是版本迭代太快，这就导致一个问题，不同版本的配置可能会不同，比如SpringCloud官网目前能看到Finchley、Edgware Dalston这几个版本，不同版本之间一定会有差异，所以这是一定值得注意的地方。  
当然除了SpringCloud的版本还要注意SpringBoot的版本，包括1.0+和2.0+的差异。  
Eureka2.0不再维护，所以注册中心可以选择Consul进行支持。

# 架构设计
最后看一下SpringCloud的架构图。（图片来源于网络）
![SpringCloud架构设计](http://dl2.iteye.com/upload/attachment/0127/5022/34c6c16a-6b3d-3095-b769-bc8ac5d02b73.jpg)