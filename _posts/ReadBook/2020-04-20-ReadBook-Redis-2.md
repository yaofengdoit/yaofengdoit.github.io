---
layout: post
title: 读书笔记之《Redis开发与运维》—— 二
category: 读书笔记
tags: [读书笔记]
---

由于内容过多，分一个系列来写，这是第二篇。

## 三、小功能大用处

### 1、慢查询分析

&ensp;&ensp;&ensp;&ensp;慢查询日志就是系统在命令执行前后计算每条命令的执行时间，当超过预设阀值，就将这条命令的相关信息记录下来。注意是命令执行
前后，redis客户端执行一条命令分为四步：发送命令、命令排队、命令执行、返回结果。所以没有慢查询不代表客户端没有超时问题。

&ensp;&ensp;&ensp;&ensp;redis提供了slowlog-log-slower-than和slowlog-max-len配置。slowlog-log-slower-than用于设置阀值，单位微秒，
默认10毫秒，也就是超过10毫秒，判定为慢查询。slowlog-max-len表示慢查询最多存储多少条，redis使用了一个列表来存储慢查询日志。当超过最大数量时，对头
的第一条数据出列。

### 2、Redis Shell

redis-cli、redis-server、redis-benchmark(可以为redis做基准性能测试)等shell工具。

### 3、Pipeline

&ensp;&ensp;&ensp;&ensp;redis提供了批量操作命令，但是大部分命令不支持批量操作。Pipeline机制能够将一组redis命令进行组装，通过一次往返时间传输
给redis，再将这组redis命令的执行结果按照顺序返回给客户端。注意：每次Pipeline组装的命令个数不要过大，否则会加大客户端的等待时间，也会造成一定的网络
阻塞，可以将包含大量命令的Pipline拆分成多次较小的Pipline来完成。

### 4、事务与Lua

&ensp;&ensp;&ensp;&ensp;redis提供了简单的事务功能以及集成Lua脚本来保证多条命令组合的原子性。将一组需要一起执行的命令放到multi和exec两个命令
之间，multi代表事务开始，exec代表事务结束，他们之间的命令原子顺序执行。如果要停止事务的执行，可以使用discard代替exec。

如果事务中的命令出现错误，如果是语法错误，会造成整个事务无法执行，如果是运行时错误，可能一部分运行成功，一部分失败。redis不支持回滚功能。如果在事务
之前，需要确保事务中的key没有被其他客户端修改过，才执行事务，否则不执行，可以使用watch命令。

redis中执行Lua脚本有两种方式：eval和evalsha。如果Lua脚本较长，可以使用redis-cli --eval执行脚本文件。--eval本质和eval一样，客户端如果想要执行
Lua脚本，首先在客户端编写好Lua脚本代码，然后把脚本作为字符串发送给服务端。使用evalsha时，首先将Lua脚本加载到Redis服务端，得到脚本的SHA1校验和，
evalsha使用SHA1作为参数可以直接执行对应的Lua脚本，避免每次发送Lua脚本的开销。这样脚本常驻服务端，脚本功能得到了复用。

script load命令将脚本内容加载到Redis内存中。

Lua可以使用redis.call、redis.pcall函数实现对redis的访问。注意：如果redis.call执行失败，脚本执行结束会直接返回错误，而redis.pcall会忽略
错误继续执行脚本。

Lua脚本功能带来的好处：<br/>
Lua脚本在Redis中是原子执行的，执行过程中不会插入其他命令。<br/>
Lua脚本可以帮助开发人员创造出自己定制的命令，并可以将这些命令常驻Redis内存中，实现复用的效果。<br/>
Lua脚本可以一次性打包多条命令，有效减少网络开销。

script flush：清除redis内存已经加载的所有Lua脚本； script kill：杀掉正在执行的Lua脚本，如果当前Lua脚本正在执行写操作，那么script kill将
不会生效。

### 5、Bitmaps

Bitmaps实际上是字符串，可以对字符串的位进行操作，可以把Bitmaps看成是一个以位为单位的数组，数组的每个单元只能存储0和1，数组的下标在Bitmaps中
叫做偏移量。

### 6、HyperLogLog

HyperLogLog并不是一种新的数据结构(实际类型为字符串类型)，而是一种基数算法，通过HyperLogLog可以利用极小的内存空间完成独立总数的统计，比如统计ID、
IP等。HyperLogLog占用内存非常小，但是需要注意存在错误率，有一定的误差。

### 7、发布订阅

和专业的消息队列系统比，redis的发布订阅略显粗糙，这里省略。

### 8、GEO

redis实现了GEO功能，用于实现基于地理位置信息的应用，底层实现是zset。


## 五、客户端

客户端与服务端之间的通信协议是在TCP协议之上构建的，Redis定制了RESP（Redis序列化协议）实现客户端和服务端的交互。

### 1、Jedis
直接和使用连接池两种。使用JedisPool不需要每次连接都生成Jedis对象，降低开销。 Jedis的Pipeline用法和Lua脚本用法。

### 2、客户端API

client list：列出与Redis服务端相连的所有客户端连接信息。
info clients：列出与redis服务端相连的客户端总数，最大缓冲区等。

输入缓冲区：Redis为每个客户端分配了输入缓冲区，将客户端发送的命令临时保存，同时Redis会从输入缓冲区拉取命令并执行。输入缓冲区会根据输入内容大小
的不同动态调整，每个客户端缓冲区的大小不能超过1G，超过后客户端将被关闭。

输出缓冲区：Redis为每个客户端分配了输出缓冲区，保存命令执行的结果返回给客户端，为redis和客户端交互返回结果提供缓冲。

输入缓冲区和输出缓冲区不会受到maxmemory限制。

monitor命令用于监控Redis正在执行的命令，注意：一旦redis的并发量较大，那么monitor客户端的输出缓冲会暴涨，可能瞬间占用大量内存。禁止生产环境
使用monitor。

