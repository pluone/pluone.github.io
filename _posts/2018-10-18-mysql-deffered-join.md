---
layout: post
title:  "利用延迟关联(deffered join)来优化limit分页查询"
date:   2018-10-18 12:00:00 +0800
categories: mysql
---

分页时需要使用limit关键字，limit的语法为`limit [offset] row_count`

limit接受1个或者2个参数，接受两个参数时第一个参数表示偏移量，即从哪一行开始取数据，第二个参数表示要取的行数。
如果只有一个参数，相当于偏移量为0  
例如:
`limit 10` 相当于`limit 0,10` 表示取前10条记录  
`limit 5,10` 表示取第6-15条记录  
当偏移量很大时，如`limit 100000,10` 取第100001-100010条记录，mysql会取出100010条记录然后将前100000条记录丢弃，这无疑是一种巨大的性能浪费。  

可以使用延迟关联的方式来进行优化  
表profiles的主键为id,且含有多列索引(sex,rating)  
原SQL语句如下  

```sql
select <cols> from profiles where sex='M' order by rating limit 100000, 10;
```

可以改写为：

```sql
select <cols> from profiles inner join
(select id form profiles where x.sex='M' order by rating limit 100000, 10)
as x using(id);
```

这里利用了覆盖索引(covering index)的特性，从多列索引(sex,rating)中找到需要的100010条数据的主键id,然后丢弃100000条id是不需要很大的开销的，然后通过关联查询找到这10条数据的列。

参考:  
《Hign Performance MySQL(3rd)》第6章Optimizing LIMIT and OFFSET
