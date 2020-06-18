---
layout: post
title:  "线上redis获取不到连接异常原因分析"
date:   2020-06-18 10:00:00 +0800
categories: 后端开发
---

redis异常，无法获取到连接，运维直接登录到redis服务器通过redis-cli连接发现没有问题。
项目重启后恢复正常。因此可怀疑是redis连接池没有可用的空闲连接，忙碌的连接又得不到释放导致的。

异常堆栈信息

```
org.springframework.data.redis.RedisConnectionFailureException: Cannot get Jedis connection; nested exception is redis.clients.jedis.exceptions.JedisException: Could not get a resource from the pool
... 中间省略
Caused by: redis.clients.jedis.exceptions.JedisException: Could not get a resource from the pool
	at redis.clients.util.Pool.getResource(Pool.java:51)
	at redis.clients.jedis.JedisPool.getResource(JedisPool.java:226)
	at redis.clients.jedis.JedisPool.getResource(JedisPool.java:16)
	at org.springframework.data.redis.connection.jedis.JedisConnectionFactory.fetchJedisConnector(JedisConnectionFactory.java:194)
	... 87 common frames omitted
Caused by: java.util.NoSuchElementException: Timeout waiting for idle object
	at org.apache.commons.pool2.impl.GenericObjectPool.borrowObject(GenericObjectPool.java:449)
	at org.apache.commons.pool2.impl.GenericObjectPool.borrowObject(GenericObjectPool.java:363)
	at redis.clients.util.Pool.getResource(Pool.java:49)
	... 90 common frames omitted
```

分析异常堆栈

`Could not get a resource from the pool` 从连接池中获取不到资源，

`Timeout waiting for idle object` 从连接池空闲队列中阻塞等待超时

那么从连接池中获取不到空闲连接的原因是什么呢，或者说为什么忙碌的连接得不到释放呢？

经过研究发现是代码的问题，jedis连接使用完毕后没有得到很好的释放，为什么说是没有得到很好的释放，而不是没有释放呢，
是因为正常来说，最正确的释放资源的方式是通过try-finally语句，在finally中释放资源，目前项目的代码则没有在finally中去释放资源，导致的问题是中间执行的业务逻辑发生了异常，然后释放资源的代码得不到执行。
具体分析如下：

项目使用redis作为分布式锁的实现方式，并且通过定义切面的方式来使用，具体伪代码如下：

```java
// 分布式锁处理的切面
public class DistributedLockAspect{

    // 错误示范，正确使用应该在finally中释放资源
    // 定义Around类型的切面
    @Around
    public void lock(ProceedingJoinPoint joinPoint){
        Jedis jedis = getConnection();
        // 获取锁
        Lock lock = getLock(jedis);
        // 处理业务逻辑
        joinPoint.proceed(); 
        // 释放锁，正确方式是在finally中释放资源
        releaseLock(jedis);
    }
}

public class FooController{

    // 使用DistributedLock注解，表示使用分布式锁
    @DistributedLock
    @RequestMapping("/api/v1/foo")
    public Object foo(){
        // 业务逻辑
    }
}
```