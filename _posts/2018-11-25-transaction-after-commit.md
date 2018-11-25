---
layout: post
title:  "Spring事务提交后执行代码"
date:   2018-11-25 16:40:00 +0800
categories: 后端开发
excerpt_separator: <!--more-->
---
在日常开发中常常会有这样的场景，在一段业务代码的最后需要发送MQ消息，但是需要事务提交后再执行发送，否则如果MQ消息消费的很快，去库中查询对应的业务数据，因为事务未提交而查询不到，导致代码报错。本文主要探讨应对这样的场景的一种方案。
<!--more-->

写法：

```java
@Transactional
public void bussiness(){
    // 处理业务
    // ---
    // 下面的代码事务提交后才执行
    TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
        @Override
        public void afterCommit() {
            //这里面是业务代码,例如发送mq
            //mqClient.send(mqMessage);
        }
    });
}

```

这段代码很简单，但是却有一个坑，afterCommit方法中不要写新的操作数据库代码，尤其不要写更新数据库的操作，因为事务不会提交，原因可以参考afterCommit方法的文档。如果确实需要在这里面执行事务提交操作，需要开启一个新的事务，写法如下：

```java
public class AnotherClass{
    // 事务的传播机制一定要写成requires_new
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void foo(){
        //业务代码
    }
}
```