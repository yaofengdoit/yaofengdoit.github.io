---
layout: post
title: 读书笔记之《Java并发编程的艺术》—— 二
category: 读书笔记
tags: [读书笔记]
no-post-nav: true
---

由于内容过多，分一个系列来写，这是第二篇。

三、Java内存模型

1、Java内存模型的基础

线程之间的通信机制有两种：共享内存和消息传递。

在Java里，所有实例域、静态域和数组元素都存储在堆内存中，堆内存在线程之间共享。局部变量、方法定义参数、异常处理参数不会在线程间共享。Java线程之间
的通信由Java内存模型(JMM)控制，JMM决定一个线程对共享变量的写入何时对另一个线程可见。线程之间的共享变量存储在主内存，每个线程都有一个私有的本地
内存(本地内存是JMM的一个抽象概念，并不真实存在)。JMM通过控制主内存与每个线程的本地内存之间的交互，来提供内存可见性保证。

执行程序时，为了提高性能，编译器和处理器常常会对指令进行重排序(编译器优化的重排序，指令级并行的重排序，内存系统的重排序)。

为了保证内存可见性，Java编译器在生成指令序列的适当位置会插入内存屏障指令来禁止特定类型的处理器重排序。JMM把内存屏障指令分为四类：LoadLoad、
StoreStore、LoadStore、StoreLoad。  StroeLoad是一个全能型的屏障，同时具有其他3个屏障的效果。

2、volatile的内存语义

volatile写的内存语义：<br/>
当写一个volatile变量时，JMM会把该线程对应的本地内存中的共享变量值刷新到主内存。当读一个volatile变量时，JMM会把该线程对应的本地内存置为无效，
线程从主内存中读取共享变量。

在每个volatile写操作的前面插入一个StoreStore屏障，在每个volatile写操作的后面插入一个StoreLoad屏障，在每个volatile读操作的后面插入一个
LoadLoad屏障，在每个volatile读操作的后面插入一个LoadStore屏障。

volatile仅仅保证对单个volatile变量的读/写具有原子性，而锁的互斥特性可以确保对整个临界区代码的执行具有原子性。

3、锁的内存语义

当线程释放锁时，JMM会把该线程对应的本地内存中的共享变量刷新到主内存中。当线程获取锁时，JMM会把该线程对应的本地内存置为无效，从而使得被监视器保护
的临界区代码必须从主内存中读取共享变量。


四、Java并发编程基础

这一部分很多内容和《Java多线程编程核心技术》重复，这里就不再记录了。


五、Java中的锁

1、Lock接口

跟synchronized的隐式获取锁相比，Lock接口在使用时显示的获取锁和释放锁。

```
public interface Lock {
    // 获取锁，调用该方法当前线程将会获取锁，当锁获得后，从该方法返回
    void lock();
    // 可中断地获取锁，和lock()方法不同的是，在该方法中会响应中断，即在锁的获取中可以中断当前线程
    void lockInterruptibly() throws InterruptedException;
    // 尝试非阻塞的获取锁，调用该方法后立即返回，如果能获取则返回true，否则返回false
    boolean tryLock();
    // 在指定的截止时间之前获取锁，如果截止时间到了仍旧没有获取锁，则返回
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    // 释放锁
    void unlock();
    // 获取等待通知组件，该组件和当前的锁绑定，当前线程只有获得了锁，才能调用该组件的wait()方法
    Condition newCondition();
}
```

2、队列同步器

队列同步器AbstractQueuedSynchronizer（AQS）是用来构建锁或其他同步组件的基础框架，它使用一个int成员变量表示同步状态，通过内置的FIFO队列
来完成资源获取线程的排队工作。

同步器的主要使用方式是继承，子类通过继承同步器并实现它的抽象方法来管理同步状态。子类推荐被定义为自定义同步组件的静态内部类，同步器自身仅仅是定义了
若干同步状态获取和释放的方法来供自定义同步组件使用。同步器即支持独占式地获取同步状态，也支持共享式地获取同步状态。

(1)同步器的设计是基于模板方法模式的，也就是说，使用者需要继承同步器并重写指定的方法，随后将同步器组合在自定义同步组件的实现中，并调用同步器实现的模板
方法，而这些模板方法将会调用使用者重写的方法。

同步器提供了3个方法来访问或修改同步状态：
```
// 同步状态，加了volatile关键字
private volatile int state;

// 获取当前同步状态
protected final int getState() {
    return state;
}

// 设置当前同步状态
protected final void setState(int newState) {
    state = newState;
}

// 使用CAS设置当前状态，该方法能够保证状态设置的原子性
protected final boolean compareAndSetState(int expect, int update) {
    // See below for intrinsics setup to support this
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

同步器可以重写的方法：
```
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}

protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
}

protected int tryAcquireShared(int arg) {
    throw new UnsupportedOperationException();
}

protected boolean tryReleaseShared(int arg) {
    throw new UnsupportedOperationException();
}

protected boolean isHeldExclusively() {
    throw new UnsupportedOperationException();
}
```




