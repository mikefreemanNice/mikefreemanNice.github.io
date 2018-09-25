---
title: dubbo交换层和传输层源码心得
date: 2018-08-22 20:34:24
comments: false
categories: rpc
tags: [dubbo]
description: 根据dubbo exchange和transport源码写一下心得体会。
---

# dubbo概览
> 以下分层结构内容来源于网络

## 服务接口层（Service）
该层是与实际业务逻辑相关的，根据服务提供方和服务消费方的业务设计对应的接口和实现。
配置层（Config）：对外配置接口，以ServiceConfig和ReferenceConfig为中心，可以直接new配置类，也可以通过spring解析配置生成配置类。
## 服务代理层（Proxy）
服务接口透明代理，生成服务的客户端Stub和服务器端Skeleton，以ServiceProxy为中心，扩展接口为ProxyFactory。
## 服务注册层（Registry）
封装服务地址的注册与发现，以服务URL为中心，扩展接口为RegistryFactory、Registry和RegistryService。可能没有服务注册中心，此时服务提供方直接暴露服务。
## 集群层（Cluster）
封装多个提供者的路由及负载均衡，并桥接注册中心，以Invoker为中心，扩展接口为Cluster、Directory、Router和LoadBalance。将多个服务提供方组合为一个服务提供方，实现对服务消费方来透明，只需要与一个服务提供方进行交互。
监控层（Monitor）：RPC调用次数和调用时间监控，以Statistics为中心，扩展接口为MonitorFactory、Monitor和MonitorService。
## 远程调用层（Protocol）
封将RPC调用，以Invocation和Result为中心，扩展接口为Protocol、Invoker和Exporter。Protocol是服务域，它是Invoker暴露和引用的主功能入口，它负责Invoker的生命周期管理。Invoker是实体域，它是Dubbo的核心模型，其它模型都向它靠扰，或转换成它，它代表一个可执行体，可向它发起invoke调用，它有可能是一个本地的实现，也可能是一个远程的实现，也可能一个集群实现。
## 信息交换层（Exchange）
封装请求响应模式，同步转异步，以Request和Response为中心，扩展接口为Exchanger、ExchangeChannel、ExchangeClient和ExchangeServer。
## 网络传输层（Transport）
抽象mina和netty为统一接口，以Message为中心，扩展接口为Channel、Transporter、Client、Server和Codec。
## 数据序列化层（Serialize）
可复用的一些工具，扩展接口为Serialization、 ObjectInput、ObjectOutput和ThreadPool

# dubbo分析
dubbo的交互层正常来讲应该是不同client端和Server端通路的维护，也就是netty中channel的维护，正常来说一个client和一个server之间相同ip和port可以有多个channel的，但是dubbo在实现知识允许一个ip:port只能维护一个通路，维护多个可能在传输效率上会有提升，但是如何保证字节传输时用相同的channel，以及rpc调用顺序如何保证是一个难点。

## 网络传输层&信息交换层
dubbo如何维护连接通路呢？首先需要知道它采用了SPI的扩展方式，所以提供了Transporter接口，实现了各种扩展，比如netty，mian，grizzly等。默认采用netty方式。

```
@SPI("netty")
public interface Transporter {

    /**
     * Bind a server.
     *
     * @param url     server url
     * @param handler
     * @return server
     * @throws RemotingException
     * @see org.apache.dubbo.remoting.Transporters#bind(URL, Receiver, ChannelHandler)
     */
    @Adaptive({Constants.SERVER_KEY, Constants.TRANSPORTER_KEY})
    Server bind(URL url, ChannelHandler handler) throws RemotingException;

    /**
     * Connect to a server.
     *
     * @param url     server url
     * @param handler
     * @return client
     * @throws RemotingException
     * @see org.apache.dubbo.remoting.Transporters#connect(URL, Receiver, ChannelListener)
     */
    @Adaptive({Constants.CLIENT_KEY, Constants.TRANSPORTER_KEY})
    Client connect(URL url, ChannelHandler handler) throws RemotingException;

}
```

dubbo捐赠给apache后增加了对netty4的支持，由之前的NettyHandler改为NettyClientHandler和NettyServerHandler。在这两个Handler中会发现他们同时维护一个NettyChannel的类:

```
final class NettyChannel extends AbstractChannel {
    private static final ConcurrentMap<Channel, NettyChannel> channelMap = new ConcurrentHashMap<Channel, NettyChannel>();
}
        
```
```
public abstract class AbstractChannel extends AbstractPeer implements Channel {
    public AbstractChannel(URL url, ChannelHandler handler) {
        super(url, handler);
    }
}
```
这个类是netty原生channel和dubbo封装的nettyChannel的映射。NettyChannel中包含了Url和ChannelHander，通过装饰者模式层层封装。读源码时很痛苦的是dubbo的send实现在哪？其实就在nettychannel中。

```
 @Override
    public void send(Object message, boolean sent) throws RemotingException {
        super.send(message, sent);

        boolean success = true;
        int timeout = 0;
        try {
            ChannelFuture future = channel.writeAndFlush(message);
            if (sent) {
                timeout = getUrl().getPositiveParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
                success = future.await(timeout);
            }
            Throwable cause = future.cause();
            if (cause != null) {
                throw cause;
            }
        } catch (Throwable e) {
            throw new RemotingException(this, "Failed to send message " + message + " to " + getRemoteAddress() + ", cause: " + e.getMessage(), e);
        }

        if (!success) {
            throw new RemotingException(this, "Failed to send message " + message + " to " + getRemoteAddress()
                    + "in timeout(" + timeout + "ms) limit");
        }
    }
```

NettyChannel的send方法具体的调用如下：
dubbo中client的请求功能的实现其实调用的是NettyChannel的send方法，但是谁调用了这个send?
NettyClient的抽象类AbstractClient实现了send(Object message,boolean sent)
先通过getChannel()获取到channel，这个channel是在dubbo包下的，再调用channel.send()方法，其中getChannel()是AbstractClient是一个抽象方法
它的实现实在NettyClient中完成的，也就是NettyChannel中io.netty.channel.channel。

这里从网上找了一张很好的图诠释了dubbo的通信过程：
![dubbo explain](http://pic.yupoo.com/kazaff/EwVU804K/78m8v.png)

