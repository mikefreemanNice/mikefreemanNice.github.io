---
title: CAS 和 Synchronized原理分析
date: 2019-02-01 11:09:26
tags: [Java]
comments: false
categories: [Java]
description: 本文着重介绍一下CAS和Synchronized的实现原理
---

# Java锁介绍
在Java并发中，我们最初接触的应该就是synchronized关键字了，但是synchronized属于重量级锁，很多时候会引起性能问题，volatile也是个不错的选择，但是volatile不能保证原子性，只能在某些场合下使用。

像synchronized这种独占锁属于悲观锁，它是在假设一定会发生冲突的，那么加锁恰好有用，除此之外，还有乐观锁，乐观锁的含义就是假设没有发生冲突，那么我正好可以进行某项操作，如果要是发生冲突呢，那我就重试直到成功，乐观锁最常见的就是CAS。

# Synchronized
## Synchronized三种使用方式
众所周知 Synchronized 关键字是解决并发问题常用解决方案，有以下三种使用方式:

- 同步普通方法，锁的是当前对象。
- 同步静态方法，锁的是当前 Class 对象。
- 同步块，锁的是 {} 中的对象。

## 实现原理
JVM 是通过进入、退出对象监视器( Monitor )来实现对方法、同步块的同步的。
具体实现是在编译之后在同步方法调用前加入一个 monitor.enter 指令，在退出方法和异常处插入 monitor.exit 的指令。
其本质就是对一个对象监视器( Monitor )进行获取，而这个获取过程具有排他性从而达到了同一时刻只能一个线程访问的目的。
而对于没有获取到锁的线程将会阻塞到方法入口处，直到获取锁的线程 monitor.exit 之后才能尝试继续获取锁。

## 原理实践
创建一个Test.java
```
public class Test {

    public static void main(String[] args) {
        Object lock = new Object();
        synchronized (lock){
            System.out.println("Synchronize");
        }
    }
}
```
执行javac Test.java 编译生成Test.class文件
执行javap -c Test.class查看具体信息

```
Compiled from "Test.java"
public class Test {
  public Test();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: new           #2                  // class java/lang/Object
       3: dup
       4: invokespecial #1                  // Method java/lang/Object."<init>":()V
       7: astore_1
       8: aload_1
       9: dup
      10: astore_2
      11: monitorenter
      12: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
      15: ldc           #4                  // String Synchronize
      17: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      20: aload_2
      21: monitorexit
      22: goto          30
      25: astore_3
      26: aload_2
      27: monitorexit
      28: aload_3
      29: athrow
      30: return
    Exception table:
       from    to  target type
          12    22    25   any
          25    28    25   any
}
```
可以看到在同步块的入口和出口分别有 monitorenter,monitorexit 指令。

## 锁优化
synchronized  很多都称之为重量锁，JDK1.6 中对 synchronized 进行了各种优化，为了能减少获取和释放锁带来的消耗引入了偏向锁和轻量锁。
### 轻量锁
当代码进入同步块时，如果同步对象为无锁状态时，当前线程会在栈帧中创建一个锁记录(Lock Record)区域，同时将锁对象的对象头中 Mark Word 拷贝到锁记录中，再尝试使用 CAS 将 Mark Word 更新为指向锁记录的指针。
如果更新成功，当前线程就获得了锁。
如果更新失败 JVM 会先检查锁对象的 Mark Word 是否指向当前线程的锁记录。
如果是则说明当前线程拥有锁对象的锁，可以直接进入同步块。
不是则说明有其他线程抢占了锁，如果存在多个线程同时竞争一把锁，轻量锁就会膨胀为重量锁。
### 解锁
轻量锁的解锁过程也是利用 CAS 来实现的，会尝试锁记录替换回锁对象的 Mark Word 。如果替换成功则说明整个同步操作完成，失败则说明有其他线程尝试获取锁，这时就会唤醒被挂起的线程(此时已经膨胀为重量锁)
轻量锁能提升性能的原因是：认为大多数锁在整个同步周期都不存在竞争，所以使用 CAS 比使用互斥开销更少。但如果锁竞争激烈，轻量锁就不但有互斥的开销，还有 CAS 的开销，甚至比重量锁更慢。
### 偏向锁
为了进一步的降低获取锁的代价，JDK1.6 之后还引入了偏向锁。
偏向锁的特征是:锁不存在多线程竞争，并且应由一个线程多次获得锁。
当线程访问同步块时，会使用 CAS 将线程 ID 更新到锁对象的 Mark Word 中，如果更新成功则获得偏向锁，并且之后每次进入这个对象锁相关的同步块时都不需要再次获取锁了。
### 释放锁
当有另外一个线程获取这个锁时，持有偏向锁的线程就会释放锁，释放时会等待全局安全点(这一时刻没有字节码运行)，接着会暂停拥有偏向锁的线程，根据锁对象目前是否被锁来判定将对象头中的 Mark Word 设置为无锁或者是轻量锁状态。
轻量锁可以提高带有同步却没有竞争的程序性能，但如果程序中大多数锁都存在竞争时，那偏向锁就起不到太大作用。可以使用 -XX:-userBiasedLocking=false 来关闭偏向锁，并默认进入轻量锁。
其他优化
### 适应性自旋
在使用 CAS 时，如果操作失败，CAS 会自旋再次尝试。由于自旋是需要消耗 CPU 资源的，所以如果长期自旋就白白浪费了 CPU。JDK1.6加入了适应性自旋:**如果某个锁自旋很少成功获得，那么下一次就会减少自旋。**

# CAS
## 实现原理
CAS底层使用JNI调用C代码实现的，如果你有Hotspot源码，那么在Unsafe.cpp里可以找到它的实现

## CAS的问题

### ABA问题
CAS需要在操作值的时候检查下值有没有发生变化，如果没有发生变化则更新，但是如果一个值原来是A，变成了B，又变成了A，那么使用CAS进行检查时会发现它的值没有发生变化，但是实际上却变化了。这就是CAS的ABA问题。常见的解决思路是使用版本号。在变量前面追加上版本号，每次变量更新的时候把版本号加一，那么A-B-A 就会变成1A-2B-3A。目前在JDK的atomic包里提供了一个类AtomicStampedReference来解决ABA问题。这个类的compareAndSet方法作用是首先检查当前引用是否等于预期引用，并且当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值。

### 循环时间长开销大
上面我们说过如果CAS不成功，则会原地自旋，如果长时间自旋会给CPU带来非常大的执行开销。

# 相关文章
- [https://juejin.im/post/5a5c488e518825733a30ae9d](https://juejin.im/post/5a5c488e518825733a30ae9d)
- [https://juejin.im/post/5a73cbbff265da4e807783f5](https://juejin.im/post/5a73cbbff265da4e807783f5)

