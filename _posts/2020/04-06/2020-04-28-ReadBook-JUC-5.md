---
layout: post
title: 读书笔记之《Java并发编程的艺术》—— 四
category: 读书笔记
tags: [读书笔记]
---

由于内容过多，分一个系列来写，这是第四篇。

## 九、Java中的线程池

线程池用于异步或并发执行任务的场景，合理使用线程池带来的好处：<br/>
(1)降低资源消耗，通过重复利用已创建的线程降低线程创建和销毁造成的消耗；<br/>
(2)提高响应速度，当任务到达时，任务可以不需要等到线程创建就能立即执行；<br/>
(3)提高线程的可管理性，使用线程池统一分配、调优和监控。


### 1、线程池的实现原理

当向线程池提交一个任务后，线程池的主要处理流程：<br/>
(1)线程池判断核心线程池里的线程是否都在执行任务，如果不是，那么创建一个新的工作线程来执行任务，如果核心线程池里的线程都在执行任务，那么进入下一步；<br/>
(2)线程池判断工作队列是否满了，如果工作队列没满，则将新提交的任务存储在这个工作队列中，如果满了，进行下一步；<br/>
(3)线程池判断线程池的线程是否都处于工作状态，如果不是，则创建一个新的工作线程来执行任务，如果满了，则交给饱和策略来处理这个任务。

ThreadPoolExecutor执行execute方法分为下面四种情况：<br/>
(1)如果当前运行的线程少于corePoolSize，那么创建新线程来执行任务(执行这一步需要获得全局锁)；<br/>
(2)如果运行的线程等于或多于corePoolSize，则将任务加入BlockingQueue；<br/>
(3)如果无法将任务加入BlockingQueue(队列已满)，则创建新的线程来处理任务(执行这一步需要获取全局锁)；<br/>
(4)如果创建新线程将使当前运行的线程超过maximumPoolSize，任务将被拒绝，并调用RejectedExecutionHandler.rejectedExecution()方法。

注意：ThreadPoolExecutor采取上述步骤的总体设计思路，是为了在执行execute()方法时，尽可能的避免获取全局锁，ThreadPoolExecutor在完成预热后(当前
运行的线程数大于等于corePoolSize)，几乎所有的execute()方法调用都在执行上面的步骤2，步骤2是不需要获得全局锁的。

工作线程：线程池创建线程时，会把线程封装成工作线程Worker，Worker在执行完任务后，还会循环获取工作队列里的任务来执行。


### 2、线程池的创建

创建线程池的几个参数：<br/>
(1)corePoolSize(线程池的基本大小)：当提交一个任务到线程池时，线程池会创建一个线程来执行任务，即使其他空闲的基本线程能够执行新任务也会
创建线程，等到需要执行的任务数大于corePoolSize时就不在创建。调用prestartAllCoreThreads()方法，线程池会提前创建并启动所有基本线程；<br/>
(2)workQueue(任务队列)：用于保存等待执行的任务的阻塞队列，比如ArrayBlockingQueue、LinkedBlockingQueue、SynchronousQueue、
PriorityBlockingQueue；<br/>
(3)maximumPoolSize(线程池最大数量)：线程池允许创建的最大线程数，如果队列满了，并且已创建的线程数小于最大线程数，则线程池会再创建
新的线程执行任务。注意：如果使用了无界的任务队列，这个参数就没效果了；<br/>
(4)threadFactory：用于创建线程的工厂，可以通过线程工厂给每个创建出来的线程设置更有意义的名字；<br/>
(5)handler(饱和策略)：当队列和线程池都满了，说明线程池处于饱和状态，那么必须采用一种策略处理提交的新任务，默认的策略是AbortPolicy——直接抛出异常，
还有其他三种策略，分别是CallerRunsPolicy——只用调用者所在线程来运行任务，DiscardPolicy——不处理，丢弃掉，DiscardOldestPolicy——丢弃队列里最前面
的任务，并执行当前任务；<br/>
(6)keepAliveTime(多余空闲线程的最长存活时间)：当线程池中线程数量大于corePoolSize，会根据keepAliveTime的值进行活性检查，一旦超时便销毁
 大于 corePoolSize 小于等于 maximumPoolSize 的线程；<br/>
(7)unit：keepAliveTime的时间单位。


### 3、向线程池提交任务

可以使用两个方法向线程池提交任务，分别是execute()和submit()方法。execute()用于提交不需要返回值的任务，无法判断任务是否被线程池执行成功。
submit()用于提交需要返回值的任务，线程池会返回一个future类型的对象，通过future对象可以判断任务是否执行成功。


### 4、关闭线程池

可以通过调用线程池里的shutdown()或shutdownNow()方法来关闭线程池。
```
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(SHUTDOWN);
        interruptIdleWorkers();
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
}

public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(STOP);
        interruptWorkers();
        tasks = drainQueue();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}
```


### 5、合理分配线程池

根据不同任务特性，合理分配线程池，可以从这几个角度分析：任务的类型(CPU密集型、IO密集型、混合型任务)、任务的优先级(高、中、低)、
任务的执行时间(长、中、短)、任务的依赖性(是否依赖其他系统资源，比如数据库连接)。


### 6、线程池的监控

对线程池进行监控，方便出问题时候，可以根据线程池的使用状况快速定位问题。


## 十、Executor框架


