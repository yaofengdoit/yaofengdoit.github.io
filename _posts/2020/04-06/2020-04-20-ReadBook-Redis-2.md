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




3、Pipeline

4、事务与Lua

5、Bitmaps

6、HyperLogLog

7、发布订阅


五、客户端