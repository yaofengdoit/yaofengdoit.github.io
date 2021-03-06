---
layout: post
title: mysql基础知识
category: MySql
tags: [MySql]
---

前言：本文主要总结一下mysql常见的基础知识，内容比较基础，对于更多深入的内容，后面会写专门系列。

### 1.MySQL本身实际上是一个SQL接口，它的内部包含了多种数据引擎，常用的包括：
<br/>
InnoDB：由Innobase Oy公司开发，支持事务；
<br/>
MyISAM：MySQL早期集成的默认数据库引擎，不支持事务。

&ensp;&ensp;&ensp;&ensp;MySQL接口和数据库引擎的关系就好比浏览器和浏览器引擎的关系。切换浏览器引擎不影响浏览器界面，切换MySQL引擎不影响自己写的应用程序使用MySQL的接口。

### 2.主键是关系表中记录的唯一标识

&ensp;&ensp;&ensp;&ensp;选取主键的一个基本原则是：不使用任何业务相关的字段作为主键。主键不应该允许NULL，应该使用BIGINT自增或者GUID类型。允许通过多个字段唯一标识记录，即两个或更多的字段都设置为主键，
这种主键被称为联合主键。对于联合主键，允许一列有重复，只要不是所有主键列都重复即可.没有必要的情况下，尽量不使用联合主键，因为它给关系表带来了复杂度的上升。

### 3.外键并不是通过列名实现的，而是通过定义外键约束实现的。
  
&ensp;&ensp;&ensp;&ensp;通过定义外键约束，关系数据库可以保证无法插入无效的数据。由于外键约束会降低数据库的性能，大部分互联网应用程序为了追求速度，并不设置外键约束，而是仅靠应用程序自身来保证逻辑的正确性。

### 4.索引是关系数据库中对某一列或多个列的值进行预排序的数据结构

&ensp;&ensp;&ensp;&ensp;通过使用索引，可以让数据库系统不必扫描整个表，而是直接定位到符合条件的记录，这样就大大加快了查询速度。索引的效率取决于索引列的值是否散列，
即该列的值如果越互不相同，那么索引效率越高。可以对一张表创建多个索引。索引的优点是提高了查询效率，缺点是在插入、更新和删除记录时，需要同时修改索引。
因此，索引越多，插入、更新和删除记录的速度就越慢。使用主键索引的效率是最高的，因为主键会保证绝对唯一。

### 5.SELECT语句可以不带FROM

&ensp;&ensp;&ensp;&ensp;不带FROM子句的SELECT语句有一个有用的用途，就是用来判断当前到数据库的连接是否有效。许多检测工具会执行一条SELECT 1，来测试数据库连接。

### 6.ORDER BY 后面跟多个条件

&ensp;&ensp;&ensp;&ensp;例如：ORDER BY age DESC, gender  表示先按age列倒序，如果有相同的age，再按gender列排序。如果有WHERE子句，那么ORDER BY子句要放到WHERE子句后面。

### 7.LIMIT ... OFFSET ...

&ensp;&ensp;&ensp;&ensp;例如：LIMIT 6 OFFSET 0表示，对结果集从0号记录开始，最多取6条。SQL记录集的索引从0开始。
OFFSET是可选的，如果只写LIMIT 6，那么相当于LIMIT 6 OFFSET 0。
在MySQL中，LIMIT 30 OFFSET 10 可以简写成LIMIT 10, 30。
使用LIMIT <M> OFFSET <N>分页时，随着N越来越大，查询效率也会越来越低。

### 8.聚合查询

&ensp;&ensp;&ensp;&ensp;使用聚合函数进行查询，就是聚合查询，它可以快速获得结果。
如果聚合查询的WHERE条件没有匹配到任何行，COUNT()会返回0，而SUM()、AVG()、MAX()和MIN()会返回NULL。

### 9.插入或替换

```
REPLACE INTO students (id, class_id, name, gender, score) VALUES (1, 1, '小明', 'F', 99);
```
若id=1的记录不存在，REPLACE语句将插入新记录，否则，当前id=1的记录将被删除，然后再插入新记录。

### 10.插入或更新

```
INSERT INTO students (id, class_id, name, gender, score) VALUES (1, 1, '小明', 'F', 99) ON DUPLICATE KEY UPDATE name='小明', gender='F', score=99;
```
插入一条新记录（INSERT），但如果记录已经存在，就更新该记录

### 11.插入或忽略

```
INSERT IGNORE INTO students (id, class_id, name, gender, score) VALUES (1, 1, '小明', 'F', 99);
```
插入一条新记录（INSERT），如果记录已经存在，就直接忽略。

### 12.FORCE INDEX

&ensp;&ensp;&ensp;&ensp;在查询的时候，数据库系统会自动分析查询语句，并选择一个最合适的索引。但是数据库系统的查询优化器并不一定总是能使用最优索引。
如果已经知道如何选择索引，可以使用FORCE INDEX强制查询使用指定的索引，前提是索引必须存在。

### 13.数据库事务

&ensp;&ensp;&ensp;&ensp;把多条语句作为一个整体进行操作的功能，被称为数据库事务。数据库事务可以确保该事务范围内的所有操作都可以全部成功或者全部失败。
如果事务失败，那么效果就和没有执行这些SQL一样，不会对数据库数据有任何改动。数据库事务具有ACID这4个特性：
<br/>
Atomic，原子性，将所有SQL作为原子工作单元执行，要么全部执行，要么全部不执行；
<br/>
Consistent，一致性，事务完成后，所有数据的状态都是一致的，即A账户只要减去了100，B账户则必定加上了100；
<br/>
Isolation，隔离性，如果有多个事务并发执行，每个事务作出的修改必须与其他事务隔离；
<br/>
Duration，持久性，即事务完成后，对数据库数据的修改被持久化存储。

### 14.隐式事务、显式事务

&ensp;&ensp;&ensp;&ensp;对于单条SQL语句，数据库系统自动将其作为一个事务执行，这种事务被称为隐式事务。
要手动把多条SQL语句作为一个事务执行，使用BEGIN开启一个事务，使用COMMIT提交一个事务，这种事务被称为显式事务。

### 15.隔离级别

&ensp;&ensp;&ensp;&ensp;对于两个并发执行的事务，如果涉及到操作同一条记录的时候，可能会发生问题。因为并发操作会带来数据的不一致性，包括脏读、不可重复读、幻读等。
数据库系统提供了隔离级别，针对性地选择事务的隔离级别，避免数据不一致的问题。
<br/>
Read Uncommitted是隔离级别最低的一种事务级别。在这种隔离级别下，一个事务会读到另一个事务更新后但未提交的数据，如果另一个事务回滚，
那么当前事务读到的数据就是脏数据，这就是脏读。
<br/>
在Read Committed隔离级别下，一个事务可能会遇到不可重复读的问题。不可重复读是指，在一个事务内，多次读同一数据，在这个事务还没有结束时，
如果另一个事务恰好修改了这个数据，那么，在第一个事务中，两次读取的数据就可能不一致。在Read Committed隔离级别下，事务不可重复读同一条记录，
因为很可能读到的结果不一致。
<br/>
在Repeatable Read隔离级别下，一个事务可能会遇到幻读（Phantom Read）的问题。幻读是指，在一个事务中，第一次查询某条记录，发现没有，
但是，当试图更新这条不存在的记录时，竟然能成功，并且，再次读取同一条记录，它就神奇地出现了。
幻读就是没有读到的记录，以为不存在，但其实是可以更新成功的，并且，更新成功后，再次读取，就出现了。
<br/>
Serializable是最严格的隔离级别。在Serializable隔离级别下，所有事务按照次序依次执行，因此，脏读、不可重复读、幻读都不会出现。
虽然Serializable隔离级别下的事务具有最高的安全性，但是，由于事务是串行执行，所以效率会大大下降，应用程序的性能会急剧降低。
如果没有特别重要的情景，一般都不会使用Serializable隔离级别。
<br/>
默认隔离级别，如果没有指定隔离级别，数据库就会使用默认的隔离级别。在MySQL中，如果使用InnoDB，默认的隔离级别是Repeatable Read。
