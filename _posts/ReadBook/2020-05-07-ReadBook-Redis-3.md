---
layout: post
title: 读书笔记之《Redis开发与运维》—— 三
category: 读书笔记
tags: [读书笔记]
---

由于内容过多，分一个系列来写，这是第三篇。

## 五、持久化

&ensp;&ensp;&ensp;&ensp;持久化功能有效的避免因进程退出造成的数据丢失问题，当下次重启时利用之前持久化的文件即可实现数据恢复。

### 1、RDB

RDB持久化是把当前进程数据生成快照保存在硬盘的过程，触发RDB持久化过程分为手动触发和自动触发。手动触发的命令有：save和bgsave命令。
save命令会阻塞当前Redis服务器，直到RDB过程完成为止，对于内存比较大的实例会造成长时间阻塞，线上不要用。bgsave命令，Redis进程执
行fork操作创建子进程，RDB持久化过程由子进程负责，完成后自动结束。阻塞只发生在fork阶段，时间很短。

RDB的优点：<br/>
RDB是一个紧凑压缩的二进制文件，代表Redis在某个时间点上的数据快照。适合于备份、全量复制的场景；</br>
Redis加载RDB恢复数据远快于AOF的方式。

RDB的缺点：</br>
RDB方式数据没法做到实时持久化、秒级持久化，因为bgsave每次运行都要执行fork操作创建子进程，属于重量级操作，频繁执行成本过高；</br>
RDB文件使用特定二进制格式保存，Redis版本演进过程中有多个格式的RDB版本，存在老版本Redis无法兼容新版本RDB格式的问题。

### 2、AOF

AOF持久化以独立日志的方式记录每次写命令，重启时再重新执行AOF文件中的命令达到恢复数据的目的，AOF的主要目的是解决了数据持久化的实时性。

AOF的工作流程操作：命令写入、文件同步、文件重写、重启加载。所有的写入命令会追加到aof_buf缓冲区中，缓存区根据对应的策略向硬盘做同步
操作（同步策略建议使用everysec），随着AOF文件越来越大，需要定期对AOF文件进行重写，达到压缩的目的，当Redis服务器重启时，可以加载AOF
文件进行数据恢复。


## 六、复制

本章大部分是运维层面的知识。简要整理下重点内容：<br/>
Redis通过复制功能实现主节点的多个副本，从节点可以灵活的通过slaveof命令建立或断开复制流程；<br/>
复制支持树状结构，从节点可以复制另一个从节点，实现一层层向下的复制流，复制分为全量复制和部分复制；<br/>
主从节点之间维护心跳和偏移量检查机制，保证主从节点通信正常和数据一致；<br/>
Redis为了保证高性能复制过程是异步的，写命令处理完后直接返回给客户端，不等待从节点复制完成，因此从节点数据集会有延迟。


## 七、阻塞

本章主要也是运维层面的知识。

Redis是典型的单线程架构，所有的读写操作都是在一条主线程中完成的，当Redis用于高并发场景时，这条主线程就变成了它的生命线。导致阻塞问题
的场景大致分为内在原因和外在原因：<br/>
内在原因：不合理的使用API或数据结构、CPU饱和、持久化阻塞等；<br/>
外在原因：CPU竞争、内存交换、网络问题等。


## 八、理解内存

### 1、内存消耗

info memory 获取内存相关指标

redis进程内消耗主要包括：自身内存+对象内存+缓冲内存+内存碎片   对象内存是Redis内存占用最大的一块。

### 2、内存管理

maxmemory 参数限制最大可用内存。

Redis采用惰性删除和定时任务删除过期建的内存回收策略：
- 惰性删除：访问时才判断是否过期，存在内存泄漏问题，当键过期一值未回收，导致内存不能及时释放
- 定时任务删除：内部维护了一个定时任务，默认每秒运行十次，根据键的过期比例（25%），使用快、慢模式回收键（快慢模式内部删除逻辑相同，只是执行
超时时间不同）

当Redis所用内存达到maxmemory时，会触发内存溢出控制策略：
- noeviction 默认的，不删除，拒绝写入，报错
- volatile-lru LRU算法删除设置了超时属性的键
- allkeys-lru LRU算法删除键
- allkeys-random 随机删除所有键
- volatile-random 随机删除过期键
- volatile-ttl 根据键值对象的ttl（time to live）属性，删除最近将要过期数据 

### 3、内存优化

- 缩减键值对象
- 共享对象池（与maxmemory+LRU冲突，注意）
- 字符串优化
- 编码优化
- 控制键的数量


## 九、哨兵

当主从模式出现故障时，Redis哨兵能自动完成故障发现和故障转移。哨兵本身是独立的Redis节点，不储存数据，只支持部分命令。


## 十、集群

redis cluster 是Redis的分布式解决方案，数据分区是分布式存储的核心。Redis集群的数据分区采用虚拟槽方式，所有的键映射到16384个槽中。
集群内部节点通信采用Gossip协议。


## 十一、缓存设计

- 缓存的使用带来的收益是能够加速读写，降低后端存储负载，带来的成本是缓存和存储数据不一致性，代码维护成本加大。

- 缓存更新策略根据具体场景，低一致性业务建议配置最大内存和淘汰策略的方式，高一致性业务可以结合使用超时剔除和主动更新，这样可以保证即使主动更新
出现了问题，也能保证数据过期时间后删除脏数据。

- 缓存粒度分为全部数据和部分数据，全部数据通用性高，但是占用空间大，部分数据通用性低，占空间小，但是业务更改的话(比如多缓存个字段),那么处理
起来复杂。

- 缓存穿透是指查询一个根本不存在的数据，缓存层和存储层都不会命中，缓存穿透将导致不存在的数据每次请求都到达了存储层，失去了缓存层保护后端存储
的意义，可能使后端存储负载加大。解决方式：（1）当存储层不命中，将空对象保留缓存层，设置一个过期时间；（2）布隆过滤器拦截。

- 无底洞优化问题，解决方案：串行命令、串行IO、并行IO、hash_tag。根据不同业务要求采用不同方案。

- 缓存雪崩优化：如果缓存层由于某些原因不能提供服务，那么所有的请求都将到达存储层，造成存储层调用量暴增。解决方案有：把缓存层设计成高可用的；
依赖隔离组件(比如Hystrix)限流降级。

- 热点key重建优化：热点key并大量很大，缓存失效的瞬间，会有大量线程来重建缓存，造成后端负载加大。解决方案：（1）互斥锁，只允许一个线程重建，其他
线程等待重建缓存的线程执行完，重新从缓存获取数据，特别注意，要考虑重建速度和影响；（2）永远不过期，key不设置过期时间，value设置逻辑过期时间。



