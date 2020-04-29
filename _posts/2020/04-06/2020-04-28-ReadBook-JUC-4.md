---
layout: post
title: 读书笔记之《Java并发编程的艺术》—— 三
category: 读书笔记
tags: [读书笔记]
no-post-nav: true
---

由于内容过多，分一个系列来写，这是第三篇。

## 六、Java并发容器和框架

### 1、ConcurrentHashMap的实现原理和使用

HashMap1.7、1.8在多线程并发情况下都会出现死循环。HashTable使用synchronized保证线程安全，在线程竞争激烈的情况下，效率很低。
ConcurrentHashMap1.7使用锁分段技术提升并发访问率。首先将数据分成一段一段的存储，然后给每一段数据配一把锁，当一个线程占用锁访问
其中一个段数据的时候，其他段的数据也能被其他线程访问。ConcurrentHashMap 1.8里用Synchronized + CAS 代替了 Segment。

后面写专文介绍ConcurrentHashMap1.7、1.8和HashMap1.7、1.8版本的改动和原理吧，这里略过。


### 2、ConcurrentLinkedQueue

实现一个线程安全的队列有两种方式：使用阻塞算法、使用非阻塞算法。使用阻塞算法的队列可以用一个锁(入队和出队用同一把锁)或两个锁(入队和
出队用不同的锁)等方式实现。非阻塞的实现方式可以使用循环CAS来实现。

ConcurrentLinkedQueue是一个基于链接节点的无界线程安全队列，采用先进先出的规则对节点进行排序。

HOPS的设计：并不是每次节点入队后都将tail节点更新为尾结点，也不是每次出队时都更新head节点，而是通过使用hops变量来控制并减少更新频率，
从而减少CAS的消耗。


### 3、Java中的阻塞队列

阻塞队列是一个支持两个附加操作的队列，支持阻塞的插入和移除方法。当队列满时，队列会阻塞插入元素的线程，直到队列不满；当队列为空时，获取元素
的线程会等待队列为非空。阻塞队列常用于生产者和消费者的场景，生产者是向队列添加元素的线程，消费者是从队列里取元素的线程。如果是无界阻塞队列，
队列不会出现满的情况。


### 4、Fork/Join框架

Fork/Join框架是一个用于并行执行任务的框架，是一个把大任务分割成若干个小任务，最终汇总每个小任务结果后得到大任务结果的框架。


## 七、Java中的原子操作类

Atomic包里提供了一些原子操作类，属于4种类型的原子更新方式：原子更新基本类型、原子更新数组、原子更新引用、原子更新属性(字段)。


## 八、Java中的并发工具类

JUC的并发包里提供了几个非常有用的并发工具类，CountDownLatch、CyclicBarrier、Semaphore工具提供了一种并发流程控制的手段,
Exchanger工具类提供了在线程间交换数据的一种手段。


### 1、CountDownLatch

CountDownLatch允许一个线程或者多个线程等待其他线程完成操作。join用于让当前执行线程等待join线程执行结束，原理是不停检查join线程
是否存活，如果join线程存活则让当前线程永远等待。CountDownLatch可以实现join的功能，并且比join的功能多。

CountDownLatch的构造函数接收一个int类型的参数作为计数器，如果想等待N个点完成，那么传入N。这个N个点，可以是N个线程，也可以是1个
线程里的N个执行步骤。调用CountDownLatch的countDown方法时，N就会减1。


### 2、CyclicBarrier

同步屏障CyclicBarrier，是让一组线程到达一个屏障(也叫同步点)时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程
才会继续运行。

```
public CyclicBarrier(int parties) {
    this(parties, null);
}
public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierAction;
}
```

上面默认的构造方法的参数表示屏障拦截的线程数量，每个线程调用await方法告诉CyclicBarrier我已经到达了屏障，然后当前线程被阻塞。上面第二个
构造方法用于线程到达屏障时，优先执行barrierAction。

注意：CountDownLatch的计数器只能使用一次，CyclicBarrier的计数器可以使用reset()方法重置。


### 3、Semaphore (控制并发线程数)

Semaphore(信号量)是用来控制同时访问特定资源的线程数量，通过协调各个线程，保证合理的使用公共资源。Semaphore可以用于做流量控制，比如数据库
连接。

Semaphore(int permits)构造方法传入一个整形数字，表示可用的许可证数量。比如传入10，表示允许10个线程获取许可证，即最大并发数是10。
用法：线程首先使用Semaphore的acquire()方法获取一个许可证，使用完了之后调用release()方法归还许可证。


### 4、Exchanger (线程间交换数据)

Exchanger是一个用于线程间协作的工具类，用于进行线程间的数据交换。它提供一个同步点，在这个同步点，两个线程可以交换彼此的数据，两个线程
通过exchange方法交换数据，如果第一个线程先执行exchange()方法，它会一直等待第二个线程也执行exchange()方法。当两个线程都达到了同步点，
这两个线程就可以交换数据。

```
public class ExchangerTest {

    private static final Exchanger<String> ex = new Exchanger<>();
    private static ExecutorService threadPool = Executors.newFixedThreadPool(2);

    public static void main(String[] args) {
        threadPool.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    String A = "water AQQQ";
                    String C = ex.exchange(A);
                    System.out.println(C);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        threadPool.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    String B = "water B";
                    String A = ex.exchange("Bq");
                    System.out.println("一致吗？" + A.equals(B) + " A->" + A + " B->" + B);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        threadPool.shutdown();
    }

}


输出结果：
一致吗？false A->water AQQQ B->water B
Bq

```

注意：上面例子的两条结果可以互换顺序，取决于CPU的调度。


