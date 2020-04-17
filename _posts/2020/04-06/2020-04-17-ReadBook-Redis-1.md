---
layout: post
title: 读书笔记之《Redis开发与运维》
category: 读书笔记
tags: [读书笔记]
no-post-nav: true
---

一、前言

&ensp;&ensp;&ensp;&ensp;读书笔记系列主要记录自己看的书籍中的知识点，算是一个归纳整理吧。Redis在我们的日常开发中可以说是很常用了，《Redis开发与运维》
这本书讲解了Redis开发和运维的方方面面，很系统、全面，关键是实用。特来撸撸它，记录一番。全书分为14章，下面将记录个人认为每章中重要的知识点。

二、Redis初识

&ensp;&ensp;&ensp;&ensp;Redis是一种基于键值对（key-value）的NoSQL数据库，Redis中的值可以是由string(字符串)、hash(哈希)、list(列表)、set(集合)、zset(有序集合)、
Bitmaps(位图)、HyperLogLog、GEO(地理信息定位)等多种数据结构和算法构成，可以满足很多的应用场景。因为Redis会将所有数据都放在内存里，所以它的读写
性能非常好。Redis还可以将内存的数据利用快照和日志的形式保存到硬盘上，这样发生断电或者机器故障，内存中的数据就不会丢失。当然Redis还提供了其他很多附加功能。
