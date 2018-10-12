---
title: 系统中限流的简单实现
date: 2018-10-12 15:00:23
tags: [Java]
comments: false
categories: [Java]
description: 如何实现简单的限流功能
---

# 问题引入
针对于系统中调用流量较高，但是一些流量可以舍弃的条件下需要限流的方式来减少对服务器的压力，大多服务中间件都会提供限流方案，近期阿里开源了Sentinel：[https://github.com/alibaba/Sentinel](https://github.com/alibaba/Sentinel)，有兴趣的同学可以关注下。
还有一个简单的限流工具推荐下： [https://github.com/wangzheng0822/ratelimiter4j](https://github.com/wangzheng0822/ratelimiter4j)。

# 限流常用算法
## 令牌桶限流
令牌桶是一个存放固定容量令牌的桶，按照固定速率往桶里添加令牌，填满了就丢弃令牌，请求是否被处理要看桶中令牌是否足够，当令牌数减为零时则拒绝新的请求。令牌桶允许一定程度突发流量，只要有令牌就可以处理，支持一次拿多个令牌。令牌桶中装的是令牌。

## 漏桶限流
漏桶一个固定容量的漏桶，按照固定常量速率流出请求，流入请求速率任意，当流入的请求数累积到漏桶容量时，则新流入的请求被拒绝。漏桶可以看做是一个具有固定容量、固定流出速率的队列，漏桶限制的是请求的流出速率。漏桶中装的是请求。

## 计数器限流
有时我们还会使用计数器来进行限流，主要用来限制一定时间内的总并发数，比如数据库连接池、线程池、秒杀的并发数；计数器限流只要一定时间内的总请求数超过设定的阀值则进行限流，是一种简单粗暴的总数量限流，而不是平均速率限流。

# 实现
通过信号量简单实现以下漏铜限流的算法

```
public class FlowLimit {

    private final static Semaphore semaphore = new Semaphore(10, true);
    private final static ExecutorService executorService = new ThreadPoolExecutor(50, 50, 0L, TimeUnit.MILLISECONDS,
            new LinkedBlockingQueue<>());
    private final static CyclicBarrier cyclicBarrier = new CyclicBarrier(50);

    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < 50; i++) {
            Task task = new Task(i);
            executorService.execute(task);
        }
        executorService.shutdown();
        System.out.println("availablePermits: " + semaphore.availablePermits());
    }

    private static boolean doExecute(int name) {
        try {
            boolean result = semaphore.tryAcquire(50, TimeUnit.MILLISECONDS);
            if (!result) {
                System.out.println(name + " :post failure");
                return false;
            }
            TimeUnit.MILLISECONDS.sleep(2000);
            System.out.println(name + " :post success");
            semaphore.release();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return true;
    }

    static class Task implements Runnable {

        private int name;

        public Task(int name) {
            this.name = name;
        }

        @Override
        public void run() {
            try {
                cyclicBarrier.await();
            } catch (InterruptedException | BrokenBarrierException e) {
                e.printStackTrace();
            }
            doExecute(name);
        }
    }
}
```
通过CyclicBarrier模拟50个并发请求，每个请求的处理时间模拟为2s，限流的大小是10，这样的结果是10个成功，其余40个失败。

顺便说一下模拟并发采用countDownLatch也可以实现，方式如下

```
public class FlowLimit {


    private final static Semaphore semaphore = new Semaphore(10, true);
    private final static CountDownLatch countDownLatch = new CountDownLatch(50);
    private final static ExecutorService executorService = new ThreadPoolExecutor(100, 100, 0L, TimeUnit.MILLISECONDS,
            new LinkedBlockingQueue<>());

    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < 50; i++) {
            Task task = new Task(i);
            executorService.execute(task);
            countDownLatch.countDown();
        }
        executorService.shutdown();
        TimeUnit.MILLISECONDS.sleep(5000);
    }

    private static boolean doExecute(int name) {
        try {
            boolean result = semaphore.tryAcquire(50, TimeUnit.MILLISECONDS);
            if (!result) {
                System.out.println(name + " :post failure");
                return false;
            }
            TimeUnit.MILLISECONDS.sleep(2000);
            System.out.println(name + " :post success");
            semaphore.release();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return true;
    }

    static class Task implements Runnable {

        private int name;

        public Task(int name) {
            this.name = name;
        }

        @Override
        public void run() {
            try {
                countDownLatch.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            doExecute(name);
        }
    }
}
```