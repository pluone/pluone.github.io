---
layout: post
title:  "理解mysql的临时表和文件排序"
date:   2019-02-24 21:45:00 +0800
categories: mysql
excerpt_separator: <!--more-->
---

我们经常看到Mysql的explain语句执行结果Extra字段有using temporary或者using filesort，本文主要是为了理解这两个短语的含义，从而有助于我们进行SQL语句优化。
<!--more-->

## 什么是临时表(temporary table)

顾名思义，临时表也是一张表，只不过不是持久的，当会话结束，临时表就会被删除掉。

## 什么是文件排序(filesort)

文件排序是相对于索引排序而言的，当不能使用索引生成排序结果的时候，mysql需要进行文件排序，排序可能是在内存中进行的，也可能需要磁盘文件的辅助。当待排序的数据量小于排序缓冲区（sort buffer）的大小时，排序直接在内存中进行，否则就需要磁盘文件的辅助进行排序，无论是哪种情况，mysql并没有进行区分，统一使用filesort来表示。


## mysql的关联查询是如何执行的

mysql对所有的关联查询都是通过循环嵌套来执行的。简单的讲，表1位于外层循环，表2位于内层循环，每次首先从外层循环中取出一条数据，然后在内存循环中进行匹配，匹配到的作为结果进行返回。下面的语句及其伪代码表示了实际的执行过程。

```sql
-- 示例语句
SELECT tbl1.col1, tbl2.col2
FROM tbl1 INNER JOIN tbl2 USING(col3)
WHERE tbl1.col1 IN(5,6);
```

```sql
-- 示例语句执行伪代码
outer_iter = iterator over tbl1 where col1 IN(5,6)
outer_row = outer_iter.next
while outer_row
    inner_iter = iterator over tbl2 where col3 = outer_row.col3
    inner_row = inner_iter.next
    while inner_row
        output [ outer_row.col1, inner_row.col2 ]
        inner_row = inner_iter.next
    end
    outer_row = outer_iter.next
end
```

## 关联查询order by执行的两种情况

关联查询中如果有order by排序语句，排序的过程会分为两种情况。  
第一种，order by语句中排序的列全部都出现在表1中，那么mysql在关联处理第一个表时就会进行文件排序，也就是在执行关联查询的外层循环时就进行排序，此时，mysql的explain结果中可以看到extra字段会看到`using filesort`.  
第二种，除了第一种情况之外，mysql都会先将关联查询的结果存放到一张临时表中，然后在所有的关联都结束后，再进行文件排序，此时mysql的explain语句的extra字段就会看到`using temporary;using filesort`.
