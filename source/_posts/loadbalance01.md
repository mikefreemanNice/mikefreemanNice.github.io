---
title: Ribbon负载均衡
date: 2018-06-11 22:56:10
tags: [SpringCloud]
comments: true
categories: [SpringCloud]
description: 负载均衡常用的两种方式
---

# 负载均衡

- 第一种是独立的进程单元，通过负载均衡策略将请求转发到不同的执行单元上，例如nginx
- 第二种是将负载均衡逻辑封装到服务的消费者上，消费者客户端维护了一个服务提供者的信息列表，通过负载均衡策略分摊给不同的服务提供者

# Ribbon负载均衡策略
- BestAvailableRule 选择最小请求数
- ClientConfigEnabledRoundRobinRule 轮询
- RandomRule 随机选择一个Server
- RoundRobinRule 轮询选择server
- RetryRule 根据轮询的方式重试
- WeightedResponseTimeRule 根据响应时间去分配一个weight，weight越低，被选择的可能性越低
- ZoneAvoidanceRule 根据server的zone区域和可用性来轮徐选择

# Ribbon负载均衡方式
Ribbon采用第二种，如何实现？
服务消费客户端从提供者拉去信息列表，缓存到本地，并且每10秒（源码）去ping一下服务端，如果ping通，不拉取，反之则重新拉取

源码如下：

```
/** * Class that contains the mechanism to "ping" all the instances *  * @author stonse * */class Pinger {

    private final IPingStrategy pingerStrategy;

    public Pinger(IPingStrategy pingerStrategy) {
        this.pingerStrategy = pingerStrategy;
    }

    public void runPinger() throws Exception {
        if (!pingInProgress.compareAndSet(false, true)) { 
            return; // Ping in progress - nothing to do        }
        
        // we are "in" - we get to Ping
        Server[] allServers = null;
        boolean[] results = null;

        Lock allLock = null;
        Lock upLock = null;

        try {
            /*             * The readLock should be free unless an addServer operation is             * going on...             */            allLock = allServerLock.readLock();
            allLock.lock();
            allServers = allServerList.toArray(new Server[allServerList.size()]);
            allLock.unlock();

            int numCandidates = allServers.length;
            results = pingerStrategy.pingServers(ping, allServers);

            final List<Server> newUpList = new ArrayList<Server>();
            final List<Server> changedServers = new ArrayList<Server>();

            for (int i = 0; i < numCandidates; i++) {
                boolean isAlive = results[i];
                Server svr = allServers[i];
                boolean oldIsAlive = svr.isAlive();

                svr.setAlive(isAlive);

                if (oldIsAlive != isAlive) {
                    changedServers.add(svr);
                    logger.debug("LoadBalancer [{}]:  Server [{}] status changed to {}", 
                      name, svr.getId(), (isAlive ? "ALIVE" : "DEAD"));
                }

                if (isAlive) {
                    newUpList.add(svr);
                }
            }
            upLock = upServerLock.writeLock();
            upLock.lock();
            upServerList = newUpList;
            upLock.unlock();

            notifyServerStatusChangeListener(changedServers);
        } finally {
            pingInProgress.set(false);
        }
    }
}
```

