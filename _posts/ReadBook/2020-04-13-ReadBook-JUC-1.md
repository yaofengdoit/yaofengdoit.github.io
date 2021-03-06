---
layout: post
title: 读书笔记之《Java多线程编程核心技术》
category: 读书笔记
tags: [读书笔记]
---

## 一、前言

&ensp;&ensp;&ensp;&ensp;读书笔记系列主要记录自己看的书籍中的知识点，算是一个归纳整理吧。《Java多线程编程核心技术》这本书主要讲解了
Java多线程相关的知识。全书分为7章，下面将记录个人认为每章中重要的知识点。

## 二、Java多线程的基础

### 1、进程和线程

进程是资源分配的最小单位，线程是CPU调度的最小单位。直观点理解：对于操作系统来说，一个任务就是一个进程，比如打开一个浏览器就是启动一个浏览器进程，
打开两个记事本就启动了两个记事本进程。有些进程不止同时干一件事，比如Word，它可以同时进行打字、拼写检查、打印等事情。在一个进程内部，要同时干多件
事，就需要同时运行多个“子任务”，把进程内的这些“子任务”称为线程（Thread）。每个进程至少要做一件事，所以进程里至少要有一个线程。线程是最小的执行
单元，而进程由至少一个线程组成。如何调度进程和线程，完全由操作系统决定。

特别注意：多进程和多线程的程序涉及到同步、数据共享的问题。进程之间共享信息可通过TCP/IP协议，线程间共享信息可通过共用内存。线程不能够独立执行，
必须依存在应用程序中，由应用程序提供多个线程执行控制。进程有独立的地址空间，相互不影响，线程没有自己独立的地址空间（地址空间都是按进程分配的，
但在地址空间里有专属于线程的线程栈，地址空间是系统给进程分配的虚拟内存，线程栈是线程自己独有的）。进程的切换比线程的切换开销大。每个进程对应一个
JVM实例，多个线程共享JVM里的堆。单核CPU执行多任务，是操作系统轮流让各个任务轮流执行，由于CPU的执行速度实在是太快了，感觉就像所有任务都在同时
执行一样。真正的并行执行多任务只能在多核CPU上实现。

注意：Java采用单线程编程模型，JVM创建主线程，主线程可以创建子线程。

### 2、Java多线程的几种实现方式

 （1）继承Thread类，重写run()方法；<br/>
 （2）实现Runnable接口，重写run()方法；<br/>
 （3）通过Callable和FutureTask创建线程；<br/>
 （4）通过线程池创建线程。

### 3、sleep()方法

使当前执行的线程休眠（暂时停止执行）指定的毫秒数，线程不会失去对监视器的所有权。休眠时间结束后，进入就绪状态，和其他线程一起竞争CPU的执行
时间。注意：wait()是Object里的方法，wait是进入线程等待池等待，出让系统资源，其他线程可以占用CPU。调用wait方法的线程，不会自己唤醒，
需要线程调用notify / notifyAll方法唤醒等待池中的所有线程，才会进入就绪队列中等待系统分配资源。sleep方法会自动唤醒，如果时间不到，
想要唤醒，可以使用interrupt方法强行打断。

### 4、停止线程

(1)使用退出标志，使线程正常退出，当run()执行完后，线程终止；<br/>
(2)stop()方法强制执行，废弃了，不要用这个方式；<br/>
(3)interrupt()方法，该方法是在当前线程打了个停止标记，并不会真的停止线程。<br/>

注意：sleep()状态下停止某一个线程，会进入catch语句，并且清除停止状态值，使之变成false。
```
/**
 * Causes the currently executing thread to sleep (temporarily cease
 * execution) for the specified number of milliseconds, subject to
 * the precision and accuracy of system timers and schedulers. The thread
 * does not lose ownership of any monitors.
 *
 * @param  millis
 *         the length of time to sleep in milliseconds
 *
 * @throws  IllegalArgumentException
 *          if the value of {@code millis} is negative
 *
 * @throws  InterruptedException
 *          if any thread has interrupted the current thread. The
 *          <i>interrupted status</i> of the current thread is
 *          cleared when this exception is thrown.
 */
public static native void sleep(long millis) throws InterruptedException;
```

### 5、interrupted()、isInterrupted()

(1)interrupted()方法，测试当前线程是否已经是中断状态，执行后具有将状态标志清除为false的功能；
(2)isInterrupted()方法，测试线程Thread对象是否已经是中断状态，但不清除状态标志。

### 6、yield()

yield()方法的作用是放弃当前的CPU资源，将它让给其他的任务去占用CPU执行时间。但是放弃的时间不确定，有可能刚刚放弃，马上又获得了CPU时间片。
yield让当前线程由“运行状态”进入到“就绪状态”。

### 7、优先级

操作系统中，线程可以划分优先级，优先级较高的线程得到的CPU资源较多。线程的优先级分为1~10 10个等级，1最低，10最高。注意：优先级和执行顺序具有
不确定性和随机性。

### 8、守护线程

Java线程分两种：用户线程、守护线程。守护线程是一种特殊的线程，当进程中不存在非守护线程了，那么守护线程自动销毁。典型的守护线程就是垃圾回收线程。


## 三、多线程中对并发访问的控制

### 1、synchronized关键字

(1)synchronized关键字取得的锁是对象锁(对象锁锁住的是，同样由synchronized修饰的方法或代码段)，而不是把一段代码或者方法、函数当作锁。
哪个线程先执行带synchronized关键字的方法，哪个线程就持有该方法所属对象的锁Lock，那么其他线程就只能呈现等待状态，前提是多个线程访问的
是同一个对象。如果多个线程访问多个对象，那么JVM会创建多个锁。

(2)synchronized关键字拥有锁重入的功能，在使用synchronized时，当一个线程得到一个对象锁后，再次请求此对象锁时是可以再次获得该对象的锁的。在一个
synchronized方法/块的内部调用本类的其他synchronized方法/块时，是永远可以得到锁的。当存在父子类继承关系时，子类是完全可以通过“可重入锁”
调用父类的同步方法的（注意：同步不可以继承，子类方法中也需要加上synchronized）。当一个线程执行的代码出现异常时，其所持有的锁会自动释放。

注意：只有共享资源的读写访问才需要同步化，如果不是共享资源，那么就没有同步的必要。

注意：A线程先持有object对象的Lock锁，那么B线程可以以异步方式调用object对象中的非synchronized类型的方法;A线程先持有object对象的Lock锁，
B线程如果这时也调用object对象的synchronized类型的方法则需等待，也就是同步。

(3)synchronized关键字声明方法的话在某些情况下是有弊端的，比如A线程调用同步方法执行一个长时间的任务，那么B线程就需要等待很长时间。这个时候可以使用
同步代码块。当两个并发线程访问同一个对象中的synchronized(this)同步代码块时，一段时间内只能有一个线程被执行，另一个线程需要等待当前线程执行完
这个代码块后才可以执行该代码块，但是另一个线程可以访问该对象中的非synchronized(this)同步代码块。

注意：当一个线程访问object的一个synchronized(this)同步代码块时，其他线程对同一个object中所有其他synchronized(this)同步代码块的访问将被
阻塞。即synchronized使用的对象监视器是一个。synchronized、synchronized(this)都是锁定当前对象的。

(4)Java还支持将“任意对象”作为“对象监视器”来实现同步，这个“任意对象”大多数是实例变量及方法的参数，使用格式为synchronized(非this对象)，
锁非this对象具有一定的优点：如果在一个类中有很多个synchronized方法，这时虽然能实现同步，但是会受到阻塞，影响效率；如果使用同步代码块锁
非this对象，则synchronized(非this)代码块中的程序与同步方法是异步的，不与其他锁this同步方法争抢this锁，则可以大大提高运行效率。

注意：当多个线程同时执行synchronized(x){}同步代码块时呈同步效果；当其他线程执行x对象中synchronized同步方法时呈同步效果；当其他线程执行
x对象方法里面的synchronized(this)代码块时也呈现同步效果。

(5)synchronized关键字加到static静态方法上是给Class类上锁，而synchronized关键字加到非static静态方法上是给对象上锁。（一个是Class锁，一个是
对象锁，是会产生异步的）。

Class锁可以对类的所有对象实例起作用，也就是说如果作用在两个实例，那么静态的同步方法还是同步运行。synchronized(类.class)同步代码块的作用和
synchronized static方法的作用是一样的。

锁对象的改变：在将任何数据类型作为同步锁时，需要注意的是，是否有多个线程同时持有锁对象，如果同时持有相同的锁对象，那么这些线程之间就是同步的，
如果分别获得锁对象，这些线程之间就是异步的。

注意：只要对象不变，即使对象的属性改变，结果还是同步的。


### 2、volatile关键字

volatile关键字的主要作用是使实例变量在多个线程间可见。volatile关键字，强制从公共堆栈中取得变量的值，而不是从线程私有数据栈中取得变量的值。
但是volatile关键字不支持原子性。

(1)synchronized和volatile比较

volatile是线程同步的轻量级实现，synchronized是重量级；volatile只能修饰变量，synchronized可以修饰方法以及代码块。<br/>
多线程访问volatile不会发生阻塞，而synchronized会发生阻塞。<br/>
volatile能保证数据的可见性，但不能保证原子性，而synchronized可以保证原子性，也可以间接保证可见性，它会将私有内存和公有内存中的数据做同步。<br/>
volatile解决的是变量在多个线程之间的可见性，而synchronized解决的是多个线程之间访问资源的同步性。


## 四、线程间的通信、交互

### 1、等待、通知机制(wait/notify机制)

在调用wait()前，线程必须获得该对象的对象级别锁，即只能在同步方法或同步块中调用wait方法。执行wait()后，当前线程释放锁。方法notify()
也要在同步方法或同步块中调用，即在调用前，线程必须获得该对象的对象级别锁。在执行notify()方法后，当前线程不会马上释放该对象锁，呈
wait状态的线程也不能马上获得该对象锁，要等到执行notify()方法的线程将程序执行完，即退出synchronized代码块后，当前线程才会释放锁。
总结：wait使线程停止运行，notify使停止的线程继续运行。

wait()方法可以使调用该方法的线程释放共享资源的锁，然后从运行状态退出，进入等待队列，直到被再次唤醒。notify()方法可以随机唤醒等待队列
中等待同一共享资源的“一个”线程，并使该线程退出等待队列，进入可运行状态。notifyAll()使所有等待队列中等待同一共享资源的“全部”线程
从等待状态退出，进入可运行状态。

wait(long)方法的功能是等待一段时间内是否有线程对锁进行唤醒，如果超过这个时间就自动唤醒。

### 2、生产者、消费者模式

原理基于wait/notify。特别注意：一些大厂面试会问这个，手写。

### 3、join()方法

join()的作用是等待线程对象销毁。使用场景举例：主线程创建并启动子线程，子线程运行时间长的话，主线程线运行完，如果主线程想要获取子线程
的运行结果，也就是主线程想要等子线程运行完再结束，那么就可以用join()。

方法join的作用是使所属的线程对象x正常执行run()方法中的任务，而使当前线程z进行无限期的阻塞，等待线程x销毁后再继续执行线程z后面的代码。

join()在内部使用wait()方法进行等待，而synchronized关键字使用的是“对象监视器”原理做同步。

### 4、ThreadLocal

ThreadLocal解决的是变量在不同线程间的隔离性，也就是不同线程拥有自己的值。（可参考：一文带你搞定ThreadLocal原理与使用）。


## 五、Lock的使用

### 1、ReentrantLock类

调用ReentrantLock对象的lock()方法获得锁，调用unlock()方法释放锁。

一个Lock对象里面可以创建多个Condition（即监视器对象）实例，线程对象可以注册在指定的Condition中，从而可以选择性的进行线程通知，
调度上更加灵活。在使用notify()/notifyAll()方法进行通知时，被通知的线程由JVM随机选择，但使用ReentrantLock结合Condition可以
实现“选择性通知”。

Object中的wait()相当于Condition里的await()。Object里的notify()相当于Condition里的signal()。

### 2、公平锁、非公平锁

锁Lock分为公平锁、非公平锁。公平锁表示线程获得锁的顺序是按照线程加锁的顺序来分配的，先来先得。非公平锁是一种获得锁的抢占机制，随机
获得锁。

### 3、ReentrantReadWriteLock

类ReentrantLock具有完全互斥排他的效果，即同一时刻只有一个线程在执行ReentrantLock.lock()方法后面的业务。这样保证了实例变量的线程
安全性，但是效率低。可以使用读写锁。

读写锁中，读相关操作的锁，称为共享锁；写操作相关的锁，称为排它锁。即读锁之间不互斥，写锁与读锁互斥，写锁与写锁互斥。


## 五、定时器

这一章都是讲Timer类的。对定时任务感兴趣的可以去研究研究分布式定时任务，实际项目中，一般还是用分布式定时任务多一些。


## 六、单例模式与多线程

单例设计模式，在实际应用中比较常见。但是结合多线程使用时候，还是需要有很多需要注意的地方。

### 1、饿汉模式/立即加载

立即加载（饿汉模式）就是使用类的时候已经将对象创建完毕。
```
public class MyObject {

    private static MyObject myObject = new MyObject();

    private MyObject() {

    }
    
    public static MyObject getInstance() {
        // 缺点：不能有其他实例变量，因为该方法没做同步，可能出现非线程安全问题
        return myObject;
    }
    
}
```

### 2、懒汉模式/延迟加载

延迟加载就是在调用get()方法时实例才被创建。
```
public class MyObject {

    private static MyObject myObject;

    private MyObject() {

    }

    public static MyObject getInstance() {
        // 没做同步，不安全
        if (myObject == null) {
            myObject = new MyObject();
        }
        return myObject;
    }

}
```

### 3、双锁检查机制（存在反射攻击问题、序列化问题）

```
public class MyObject {

    private volatile static MyObject myObject;

    private MyObject() {

    }

    public static MyObject getInstance() {
        if (myObject == null) {
            synchronized (MyObject.class) {
                if (myObject == null) {
                    myObject = new MyObject();
                }
            }
        }
        return myObject;
    }

}
```

### 4、使用静态内置类（存在序列化问题）

```
public class MyObject {

    private static class MyObjectHelper {
        private static MyObject myObject = new MyObject();
    }

    private MyObject() {

    }

    public static MyObject getInstance() {
        return MyObjectHelper.myObject;
    }

}
```

### 5、使用static代码块

静态代码块中的代码在使用类的时候就已经执行了，可以利用该特性来实现单例设计模式。
```
public class MyObject {

    private static MyObject instance = null;
    
    private MyObject() {
        
    }
    
    static {
        instance = new MyObject();
    }
    
    public static MyObject getInstance() {
        return instance;
    }
    
}
```

### 6、使用枚举enum （最佳，推荐这种）

枚举enum和静态代码块的特性相似，在使用枚举类时，构造方法会被自动调用，也可以利用该特性实现单例设计模式。
```
public enum Singleton {

    INSTANCE;

    public Singleton getInstance() {
       return INSTANCE;
    }

}
```


## 七、拾漏增补

### 1、线程的状态

```
public enum State {
    // 至今尚未启动的线程
    New,
    // 正在JVM中执行的线程
    RUNNABLE,
    // 受阻塞并等待某个监视器锁的线程
    BLOCKED,
    // 无限期的等待另一个线程来执行某一特定操作的线程
    WAITING,
    // 等待另一个线程来执行，取决于指定等待时间的操作的线程
    TIMED_WAITING,
    // 已退出的线程
    TERMINATED;
}
```

New状态是线程实例化后还未执行start()方法时的状态，Runnable包含Ready和Running，yield就是将线程从Running置为Ready。<br/>
wait、join--->WAITING，sleep(time)、wait(time)、join(time)--->TIMED_WAITING

### 2、线程组 (ThreadGroup)

可以把线程归属到线程组中，线程组中可以有线程对象，也可以有线程组，组里还可以有线程。线程组的作用是批量管理线程或者线程组对象。

线程组有自动归属特性，如果实例化一个ThreadGroup线程组x时，没有指定x所属的线程组，那么x线程组自动归属到当前线程对象所属的线程组里。

JVM的根线程组是system，system没有父线程组。(我们常用的main开始方法，它所在的线程组是main，线程组main的父线程组是system)。

ThreadGroup.interrupt()方法可以将该组中所有正在运行的线程批量停止。