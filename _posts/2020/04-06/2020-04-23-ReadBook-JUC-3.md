---
layout: post
title: 读书笔记之《Java并发编程的艺术》—— 二
category: 读书笔记
tags: [读书笔记]
no-post-nav: true
---

由于内容过多，分一个系列来写，这是第二篇。

## 三、Java内存模型

### 1、Java内存模型的基础

线程之间的通信机制有两种：共享内存和消息传递。

在Java里，所有实例域、静态域和数组元素都存储在堆内存中，堆内存在线程之间共享。局部变量、方法定义参数、异常处理参数不会在线程间共享。Java线程之间
的通信由Java内存模型(JMM)控制，JMM决定一个线程对共享变量的写入何时对另一个线程可见。线程之间的共享变量存储在主内存，每个线程都有一个私有的本地
内存(本地内存是JMM的一个抽象概念，并不真实存在)。JMM通过控制主内存与每个线程的本地内存之间的交互，来提供内存可见性保证。

执行程序时，为了提高性能，编译器和处理器常常会对指令进行重排序(编译器优化的重排序，指令级并行的重排序，内存系统的重排序)。

为了保证内存可见性，Java编译器在生成指令序列的适当位置会插入内存屏障指令来禁止特定类型的处理器重排序。JMM把内存屏障指令分为四类：LoadLoad、
StoreStore、LoadStore、StoreLoad。  StroeLoad是一个全能型的屏障，同时具有其他3个屏障的效果。


### 2、volatile的内存语义

volatile写的内存语义：<br/>
当写一个volatile变量时，JMM会把该线程对应的本地内存中的共享变量值刷新到主内存。当读一个volatile变量时，JMM会把该线程对应的本地内存置为无效，
线程从主内存中读取共享变量。

在每个volatile写操作的前面插入一个StoreStore屏障，在每个volatile写操作的后面插入一个StoreLoad屏障，在每个volatile读操作的后面插入一个
LoadLoad屏障，在每个volatile读操作的后面插入一个LoadStore屏障。

volatile仅仅保证对单个volatile变量的读/写具有原子性，而锁的互斥特性可以确保对整个临界区代码的执行具有原子性。


### 3、锁的内存语义

当线程释放锁时，JMM会把该线程对应的本地内存中的共享变量刷新到主内存中。当线程获取锁时，JMM会把该线程对应的本地内存置为无效，从而使得被监视器保护
的临界区代码必须从主内存中读取共享变量。


## 四、Java并发编程基础

这一部分很多内容和《Java多线程编程核心技术》重复，这里就不再记录了。


## 五、Java中的锁

### 1、Lock接口

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

### 2、队列同步器

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
// 独占式获取同步状态
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
// 独占式释放同步状态
protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
}
// 共享式获取同步状态
protected int tryAcquireShared(int arg) {
    throw new UnsupportedOperationException();
}
// 共享式释放同步状态
protected boolean tryReleaseShared(int arg) {
    throw new UnsupportedOperationException();
}
// 当前同步器是否在独占模式下被线程占用
protected boolean isHeldExclusively() {
    throw new UnsupportedOperationException();
}
```

实现自定义同步组件时，将会调用同步器提供的模板方法，比如下面几个模板方法：
```
// 独占式获取同步状态
public final void acquire(int arg)
// 共享式获取同步状态
public final void acquireShared(int arg)
// 独占式释放同步状态
public final boolean release(int arg)
// 共享式释放同步状态
public final boolean releaseShared(int arg)
// 获取等待在同步队列上的线程集合
public final Collection<Thread> getQueuedThreads()
```

同步器提供的模板方法基本上分为三类：独占式获取和释放同步状态、共享式获取和释放同步状态、查询同步队列中的等待线程情况。


(2)队列同步器的实现分析

同步器依赖内部的同步队列(一个FIFO双向队列)来完成同步状态的管理，当前线程获取同步状态失败时，同步器会将当前线程以及等待状态等信息构造成为一个点
(Node)并将其加入到同步队列，同时会阻塞当前线程，当同步状态释放时，会把首节点中的线程唤醒，使其再次尝试获取同步状态。

同步队列中的节点用来保存获取同步状态失败的线程引用、等待状态以及前驱和后继节点：
```
static final class Node {
    ...
    volatile int waitStatus;
    volatile Node prev;
    volatile Node next;
    volatile Thread thread;
    Node nextWaiter;
    ...
}
```

同步器拥有首节点和尾结点，没有成功获取同步状态的线程会成为节点加入到队列的尾部。头结点拥有同步状态。

独占式同步状态获取和释放过程总结：在获取同步状态时，同步器维护一个同步队列，获取状态失败的线程都会被加入到队列中并在队列中进行自旋；移出队
列(停止自旋)的条件是前驱节点为头结点且成功获取了同步状态。在释放同步状态时，同步器调用release(int arg)释放同步状态，然后唤醒头结点的
后继节点。

共享式同步状态获取和独占式同步状态获取的最主要区别在于同一时刻能否有多个线程同时获取到同步状态。共享式获取的自旋过程中，成功获取到同步状态并
退出自旋的条件就是tryAcquireShared(int arg)方法返回值大于等于0。释放调用releaseShared(int arg)，释放同步状态后，将会唤醒后续处于
等待状态的节点。释放同步状态的操作会同时来自多个线程。

独占式超时获取同步状态，可以在指定的时间段内获取同步状态。


### 3、重入锁

重入锁ReentrantLock，支持重进入的锁，它表示该锁能够支持一个线程对资源的重复加锁，该锁还支持获取锁时的公平和非公平性选择。如果在绝对时间上，先对
锁进行获取的请求一定先被满足，那么这个锁是公平的，反之，是不公平的。公平的锁机制往往没有非公平的锁机制的效率高。

重进入是指任意线程在获取到锁之后能够再次获取该锁而不会被锁所阻塞。锁需要去识别获取锁的线程是否为当前占据锁的线程，如果是，那么再次成功获取；锁最终
释放是指，线程重复n次获得了锁，随后在第n次释放该锁后，其他线程能够获得该锁。锁获取时进行计数自增，锁释放时进行计数自减，当计数等于0时表示锁已经
成功释放。


### 4、读写锁

排它锁在同一时刻只允许一个线程进行访问，而读写锁在同一时刻可以允许多个读线程访问，但是在写线程访问时，所有的读线程和其他的写线程均被阻塞(注意：
其他的写线程，当前的写线程可以重入，即写线程获取了写锁之后能够再次获取到写锁)。读写锁的性能比排它锁好，大部分场景是读多写少的，在读多写少的情况下，
读写锁能够提供比排它锁更好的并发性和吞吐量。

读写锁维护了一对锁，一个读锁一个写锁。读写锁的自定义同步器需要在同步状态(一个整形变量)上维护多个读线程和一个写线程的状态。高16位表示读，低16位表示写。

如果当前线程在获取写锁时，读锁已经被获取或者该线程不是已经获取写锁的线程，那么当前线程进入等待状态。只有等待其他读线程都释放了读锁，写锁才能被当前
线程获取，而写锁一旦被获取，则其他读写线程的后续访问均被阻塞。

在没有其他写线程访问时(写状态为0),读锁总会被成功地获取。如果当前线程在获取读锁时，写锁已被其他线程获取，则进入等待状态。如果当前线程获取了写锁或者
写锁未被获取，那么当前线程获取读锁。

锁降级：写锁降级为读锁，指的是把持住(当前拥有的)写锁，再获取到读锁，随后释放(先前拥有的)写锁的过程。


### 5、LockSupport工具

LockSupport定义了一组以park开头的方法用来阻塞当前线程，以及unpark方法唤醒一个被阻塞的线程。


### 6、Condition接口

Condition接口提供了类似Object的监视器方法，Condition对象是由Lock对象创建出来的，获取一个Condition必须通过Lock的newCondition()方法。
ConditionObject是AQS的内部类，每个Condition对象都包含着一个等待队列，这是实现等待/通知功能的关键。同步器拥有一个同步队列和多个等待队列。
同步队列和等待队列中节点类型都是AQS里的Node。


