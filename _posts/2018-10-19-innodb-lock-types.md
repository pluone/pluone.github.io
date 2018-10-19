---
layout: post
title:  "innodb引擎的锁类型"
date:   2018-10-19 12:00:00 +0800
categories: mysql
tags: mysql innodb 锁
excerpt_separator: <!--more-->
---
mysql的innodb引擎提供了对事务的支持，而这种支持和innodb提供的各种类型的锁有着紧密的关系，本文主要了解innodb提供的锁类型。
<!--more-->
## 基本锁类型

所有的锁都可以分为两类，共享锁(shared lock)和拍他锁(exclusive lock)  
共享锁(S锁)允许持有锁的事务进行读取操作  
拍他锁(X锁)允许持有锁的事务进行更新或者删除操作  

两个不同的事务t1和t2  
如果t1持有某一行r的S锁，当t2请求行r的S锁时可以立即获得，t2请求行r的X锁时必须等待r上的S锁被释放之后  

如果t1持有行r的X锁，t2请求行r的S锁和X锁均需要等待。  

## innodb锁类型

### 意图锁(intention lock)

是表级锁用来表明一个事务想要获取到表中行的哪种类型的锁(S锁还是X锁)，有IS和IX两种。
IS锁表示事务想要获取表中行的S锁
IX锁表示事务想要获取表中行的X锁
当前的表锁一种有4种，X锁，S锁，IS锁和IX锁

### 记录锁(record lock)

记录锁锁的是索引中的一条记录。同样有S锁和X锁两种。

>A record lock is a lock on an index record. For example, SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE; prevents any other transaction from inserting, updating, or deleting rows where the value of t.c1 is 10.

### 间隙锁(gap lock)

间隙锁锁的是索引之间的间隙.例如当前所有有两个值10和20.
它们之间的间隙有三种(-∞,10)
(10,20)
(20,+∞)

select c1 from t where c1 between 10 and 20 for update;这条SQL可以阻止t.c1列插入15的值。
间隙锁同样有S锁和X锁之分，但是这两种不兼容的锁却可以同时获得。间隙锁的存在纯粹是为了避免插入操作，所以不同的事务都是可以同时获取到间隙锁的，因此可以理解为间隙锁的S锁和X锁是等效的。

> A gap lock is a lock on a gap between index records, or a lock on the gap before the first or after
the last index record. For example, SELECT c1 FROM t WHERE c1 BETWEEN 10 and 20 FOR UPDATE; prevents other transactions from inserting a value of 15 into column t.c1, whether or not there was already any such value in the column, because the gaps between all existing values in the range are locked.

### next-key lock

next-key lock是记录锁和在记录之前的间隙的间隙锁的组合。

> A next-key lock is a combination of a record lock on the index record and a gap lock on the gap before the index record.

### 插入意图锁(insert intention lock)

是间隙锁的一种，insert操作时会首先获取插入意图锁，然后进行插入操作。插入意图锁表明了这样的意图，当多个事务同时执行插入操作到相同的间隙时,如果它们插入的位置是不同的那么插入可以并发执行，不需要互相等待。例如有索引记录4和7，两个不同的事务要执行5和6的插入，插入之前它们都会获得(4,7)之间的插入意图锁，但是插入操作不会阻塞，因为它们插入的是不同的值。

> An insert intention lock is a type of gap lock set by INSERT operations prior to row insertion. This lock signals the intent to insert in such a way that multiple transactions inserting into the same index gap need not wait for each other if they are not inserting at the same position within the gap. Suppose that there are index records with values of 4 and 7. Separate transactions that attempt to insert values of 5 and 6, respectively, each lock the gap between 4 and 7 with insert intention locks prior to obtaining the exclusive lock on the inserted row, but do not block each other because the rows are nonconflicting.

### 自增锁(auto-inc lock)

是一种特殊类型的表锁，对AUTO_INCREMENT列执行插入操作的事务需要获取该锁。

<<<<<<< HEAD
> An AUTO-INC lock is a special table-level lock taken by transactions inserting into tables with AUTO_INCREMENT columns. In the simplest case, if one transaction is inserting values into the table, any other transactions must wait to do their own inserts into that table, so that rows inserted by the first transaction receive consecutive primary key values.
=======
> An AUTO-INC lock is a special table-level lock taken by transactions inserting into tables with AUTO_INCREMENT columns. In the simplest case, if one transaction is inserting values into the table, any other transactions must wait to do their own inserts into that table, so that rows inserted by the first transaction receive consecutive primary key values.
>>>>>>> d789f11b1296b98400babe1556c4a00d2540b39b
