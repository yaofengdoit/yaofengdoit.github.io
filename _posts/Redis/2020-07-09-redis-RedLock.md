---
layout: post
title: RedLock
category: redis
tags: [redis]
---

## 一、What?

RedLock，即Redis Distributed Lock，使用redis实现的分布式锁。

官网文档地址如下：https://redis.io/topics/distlock

这个锁的算法实现了多redis实例的情况，相对于单redis节点来说，优点在于防止了单节点故障造成整个服务停止运行的情况。

最低保证分布式锁的有效性及安全性的要求如下：
- 互斥；任何时刻只能有一个client获取锁
- 释放死锁；即使锁定资源的服务崩溃或者分区，仍然能释放锁
- 容错性；只要多数redis节点（一半以上）在使用，client就可以获取和释放锁


## 二、How?

### 1、单实例redis实现分布式锁

redis单实例中实现分布式锁的正确方式（原子性非常重要）:

(1).设置锁时，使用set命令，因为其包含了setnx,expire的功能，起到了原子操作的效果，给key设置随机值，并且只有在key不存在时才设置成功返回True,
并且设置key的过期时间.

```
// NX 表示if not exist 就设置并返回True，否则不设置并返回False   PX 表示过期时间用毫秒级， 30000 表示这些毫秒时间后此key过期
SET key_name my_random_value NX PX 30000
```
               
(2).在获取锁后，并完成相关业务后，需要删除自己设置的锁（必须是只能删除自己设置的锁，不能删除他人设置的锁）；

删除原因：保证服务器资源的高利用效率，不用等到锁自动过期才删除；

删除方法：最好使用Lua脚本删除（redis保证执行此脚本时不执行其他操作，保证操作的原子性），代码如下；逻辑是 先获取key，如果存在并且值是自己设置
的就删除此key，否则就跳过；

```
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

### 2、多实例redis实现分布式锁(RedLock)

多实例的redis实现分布式锁，可以有效防止单点故障。

例如有5个完全独立的redis主服务器：
- 获取当前时间戳；
- client尝试按照顺序使用相同的key，value获取所有redis服务的锁，在获取锁的过程中，获取时间比锁过期时间短很多，这是为了不要过长时间等待已经
关闭的Redis服务，并且试着获取下一个redis实例；
- client通过获取所有能获取的锁后的时间减去第一步的时间，这个时间差要小于TTL(Time To Live,redis key的过期时间或有效生存时间)时间并且至少
有3个redis实例成功获取锁，才算真正的获取锁成功；
- 如果成功获取锁，则锁的真正有效时间是TTL减去第三步的时间差的时间；比如：TTL是5s,获取所有锁用了2s,则真正锁有效时间为3s(其实应该再减去时钟漂移);
- 如果客户端由于某些原因获取锁失败，便会开始解锁所有redis实例；因为可能已经获取了小于3个锁，必须释放，否则影响其他client获取锁。




