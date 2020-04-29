---
layout: post
title: 一文带你搞定ThreadLocal原理与使用
category: 多线程
tags: [多线程]
no-post-nav: true
---

## 一、前言

&ensp;&ensp;&ensp;&ensp;成员变量会产生线程安全问题，局部变量不会有线程安全问题，因为局部变量是线程私有的，而成员变量是线程共享的。高并发的时候，调用一些公有的对象资源的时候，会有线程安全的问题。
解决线程安全问题：（1）对成员变量进行加锁，这样的话其他线程要使用的话，就必须等待，耗时；（2）把成员变量变成局部方法变量，很显然，这样的话不合理，设置为局部变量，就不能在
各个方法中使用了。这个时候可以使用ThreadLocal来解决。ThreadLocal是并发场景下用来解决变量共享问题的类，它能使原本线程间共享的对象进行线程隔离，即一个对象只对一个线程可见，
特别适用于各个线程依赖不同的变量值完成操作的场景。那么ThreadLocal是怎么做到的呢？

## 二、引用类型

&ensp;&ensp;&ensp;&ensp;在分析ThreadLocal原理之前，先回顾下四种引用类型：强引用、软引用、弱引用、虚引用。对象在堆上创建之后所持有的引用是一种变量类型，
引用的可达性是JVM判断能否被垃圾回收的基本条件。

（1）强引用：日常所用的new一个对象，比如Object o = new Object();只要对象有强引用指向，并且GC Roots可达，那么java内存回收时，即使内存耗尽，
报出OutOfMemoryError，也不会回收该对象。

（2）软引用：软引用的生命周期比强引用短一些，通过SoftReference实现。在即将OOM之前，垃圾回收器会把软引用指向的对象加入回收范围，
以获得更多的内存空间。软引用一般用来实现对内存敏感的缓存，如果有空闲内存就保留缓存，内存不足时就清理掉，这样缓存的同时不会耗尽内存。

（3）弱引用：弱引用的生命周期比软引用短，通过WeakReference实现。如果弱引用指向的对象只存在弱引用这条线路，则在下一次GC时会被回收。
由于GC时间的不确定性，弱引用何时被回收也具有不确定性。ThreadLocal中的Entry就用到了弱引用。

（4）虚引用：极弱的一种引用关系，通过PhantomReference实现。任何时候都可以被GC回收，一个对象设置虚引用的目的是希望能在这个对象被回收时收到一个系统通知。
注意，虚引用必须配合ReferenceQueue使用，在垃圾回收前会把虚引用加入到引用队列中。

## 三、ThreadLocal原理

&ensp;&ensp;&ensp;&ensp;ThreadLocal提供了几个核心方法：
```
public T get()
public void set(T value)
public void remove()
```
其中：get()获取当前线程的副本变量值，set()保存当前线程的副本变量值，remove移除当前线程的副本变量值。

### 1、get()

```
public T get() {
    // 获取当前线程
    Thread t = Thread.currentThread();
    // 当前线程的threadLocals 
    ThreadLocalMap map = getMap(t);
    // threadLocals不为空，就可以在map中寻找到本地变量的值
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    // threadLocals为空的话，就初始化当前线程的threadLocals变量
    return setInitialValue();
}

private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    // 当前线程为key，去找对应的线程变量，找对应的map
    ThreadLocalMap map = getMap(t);
    if (map != null)
        // map不为null，直接添加本地变量，key为当前线程，值为添加的本地变量值
        map.set(this, value);
    else
        // map为null，首次添加，需要先创建出对应的map，new了一个ThreadLocalMap
        createMap(t, value);
    return value;
}
```

### 2、set()

```
public void set(T value) {
    Thread t = Thread.currentThread();
    // 当前线程为key，去找对应的线程变量，找对应的map
    ThreadLocalMap map = getMap(t);
    if (map != null)
        // map不为null，直接添加本地变量，key为当前线程，值为添加的本地变量值
        map.set(this, value);
    else
        // map为null，首次添加，需要先创建出对应的map，new了一个ThreadLocalMap
        createMap(t, value);
}

ThreadLocalMap getMap(Thread t) {
    // 获取线程自己的变量threadLocals，并绑定到当前调用线程的成员变量threadLocals上
    return t.threadLocals;
}

void createMap(Thread t, T firstValue) {
    // 不仅创建了threadLocals，同时也将要添加的本地变量值添加到了threadLocals中
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

每个线程都有一个ThreadLocalMap对象，每一个新的线程都会实例化一个ThreadLocalMap，并赋值给线程的成员变量threadLocals。
使用时若已经存在threadLocals则直接使用已经存在的对象。

### 3、remove()

```
public void remove() {
    // 获取当前线程绑定的threadLocals
     ThreadLocalMap m = getMap(Thread.currentThread());
    // map不为null，移除当前线程中指定ThreadLocal实例的本地变量
     if (m != null)
         m.remove(this);
 }
```

### 4、ThreadLocalMap里的Entry

```
static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
```

ThreadLocalMap是ThreadLocal的内部类，没有实现Map接口，用独立的方式实现了Map的功能，其内部的Entry也独立实现。
Entry用来保存K-V结构数据，Entry中key只能是ThreadLocal对象。注意：Entry继承自WeakReference，但只有Key是弱引用类型的，Value并非弱引用。

## 四、要点

### 1、ThreadLocal不支持继承性

同一个ThreadLocal变量在父线程中被设置值后，在子线程中是获取不到的，因为threadLocals中为当前调用线程对应的本地变量。如果子线程要访问父线程的
本地变量怎么办呢？可以使用InheritableThreadLocal来解决，InheritableThreadLocal类继承了ThreadLocal，并且重写了childValue、getMap、createMap三个方法。

### 2、Hash冲突的解决

注意到ThreadLocalMap里没有next引用，那么Hash冲突的话就不会像HashMap那样使用链地址法，而是采用开放寻址法里的线性探测法。
即根据初始key的hashcode值确定元素在table数组中的位置，如果发现这个位置上已经有其他key值的元素被占用，则利用固定的算法寻找一定步长的下个位置，
依次判断，直至找到能够存放的位置。ThreadLocalMap解决Hash冲突的方式就是简单的步长加1或减1，寻找下一个相邻的位置。

### 3、0x61c88647

```
int i = key.threadLocalHashCode & (len-1)
int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1)
```
Entry的索引i的位置是通过将threadLocalHashCode进行一个位运算（取模）得到的。threadLocalHashCode的值为什么取0x61c88647呢？
这点非常有趣，0x61c88647是斐波那契散列乘数,它的优点是通过它散列(hash)出来的结果分布会比较均匀，可以很大程度上避免hash冲突。

### 4、脏数据与内存泄漏

由于ThreadLocalMap的key是弱引用，而Value是强引用。这就导致了一个问题，ThreadLocal在没有外部对象强引用时，发生GC时弱引用Key会被回收，
而Value不会回收，如果创建ThreadLocal的线程一直持续运行，那么复用线程(线程池)就会产生脏数据。这个Entry对象中的value有可能一直得不到回收，就会发生内存泄露。
解决这两个问题的方法是，每次用完ThreadLocal，要及时调用remove()来清理。