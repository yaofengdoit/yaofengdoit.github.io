---
layout: post
title: inner join on会过滤掉两边空值的条件
category: MySql
tags: [MySql]
---

##### 前两天工作过程中，遇到一个问题，关于join on查询的，对于查出来的结果一直都很疑惑，这里记录一下。

##### 1.首先看下面这条sql查询语句：

![](https://yaofengdoit.github.io/assets/images/2019/mysql/join-1.png)

查询出来的结果是25053

##### 2.加个 o.lat = n.lat 的条件：

![](https://yaofengdoit.github.io/assets/images/2019/mysql/join-2.png)

查询出来的结果是15586

##### 3.现在我们将条件改成 o.lat ！= n.lat，查出来的结果是不是应该显示 25053-15586的差值呢？

![](https://yaofengdoit.github.io/assets/images/2019/mysql/join-3.png)

我们发现结果并不是预想的那样，而是125。奇怪，剩下的25053-15586-125 = 9342条数据哪里去了呢，怎么查询不出来？

##### 4.再看下面这条sql语句，我们过滤掉 o.lat和n.lat 都是空的情况：

![](https://yaofengdoit.github.io/assets/images/2019/mysql/join-4.png)

9342！！
<br/>
啊哈，原来如此，join on 会过滤掉两边都是空值的条件。
<br/>
如果返回左表中为null的数据，可以使用left join，相反，如果返回右表中null的数据，使用right join。
<br/>
inner join（可以简写成join），将不会返回左右表中均为null的数据。



