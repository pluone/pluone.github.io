---
layout: post
title:  "应用频繁数据库连接错误问题分析"
date:   2019-05-12 10:00:00 +0800
categories: mysql
excerpt_separator: <!--more-->
---

本文主要分析了web应用在使用过程中频繁报数据库连接异常的原因，以及解决方案。
<!--more-->

## 问题描述

从前端页面请求后端接口，页面拿不到任何数据，后台一直报相同的错误，重启应用可以临时解决该问题，异常堆栈如下：

```java
org.springframework.transaction.CannotCreateTransactionException: Could not open JDBC Connection for transaction; nested exception is com.mysql.jdbc.exceptions.jdbc4.MySQLNonTransientConnectionException: No operations allowed after connection closed.

Caused by: com.mysql.jdbc.exceptions.jdbc4.MySQLNonTransientConnectionException: No operations allowed after connection closed.

Caused by: com.mysql.jdbc.exceptions.jdbc4.CommunicationsException: Communications link failure

The last packet successfully received from the server was 2,246,094 milliseconds ago.  The last packet sent successfully to the server was 18,119 milliseconds ago.

Caused by: java.net.SocketException: Operation timed out (Read failed)
```

## 问题分析

从异常堆栈初步可以看出，是数据库连接已经关闭导致的，至于为什么每次请求都是这样的错误，经过认真分析，得出如下结论：
项目中配置了tomcat-jdbc连接池，可能是被关闭的连接没有被销毁，又被重新放入到了连接池中，然后下一次请求到来的时候，又从连接池中拿到了这个异常的连接，所以应用持续的报相同的错误。

<img src="/assets/db-conn-pool.png" alt="数据库连接池" style="width: 500px;"/>

## 解决方案

tomcat-jdbc连接池的配置提供了testOnBorrow参数，根据文档这个参数的含义是：当从连接池获取连接时进行验证，如果验证失败，则丢弃该连接，然后获取新的连接。这个参数需要和validationQuery参数配合使用，validationQuery参数指定了验证的语句，例如select 1

```java
    // 文档来源org.apache.tomcat.jdbc.pool.PoolConfiguration
    /**
     * The indication of whether objects will be validated before being borrowed from the pool.
     * If the object fails to validate, it will be dropped from the pool, and we will attempt to borrow another.
     * NOTE - for a true value to have any effect, the validationQuery parameter must be set to a non-null string.
     * Default value is false
     * In order to have a more efficient validation, see {@link #setValidationInterval(long)}
     * @param testOnBorrow set to true if validation should take place before a connection is handed out to the application
     * @see #getValidationInterval()
     */
    public void setTestOnBorrow(boolean testOnBorrow);
```