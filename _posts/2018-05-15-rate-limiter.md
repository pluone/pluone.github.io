---
layout: post
title:  "一个简单的限流器"
date:   2018-05-15 13:40:00 +0800
categories: code
---

**需求**: 一个类似于商品秒杀的系统,商品的列表页面和详情页面进行了缓存,可能在短时间内有大量的请求进入,正常情况下每一次购买操作都会触发一次异步的缓存刷新操作,但是秒杀的时候,会在短时间内产生大量的缓存更新请求,而这些请求其实是没有必要全部处理的,此时可以采用限流的方式控制缓存刷新的频率

下面是一个简单的限流器,可以限定每`per`秒内能够处理的最大请求数`rate`,对于过多的请求则直接丢弃

```java
class RateLimiter {
    private final int rate;
    private final int per;
    private double allowance;
    private Long lastCheck;

    RateLimiter(int rate, int per) {
        this.rate = rate;
        this.per = per;
        this.allowance = rate;
        lastCheck = System.currentTimeMillis();
    }

    synchronized void rateLimit(UpdateCacheEvent cacheEvent) {
        Long current = System.currentTimeMillis();
        Long timePassed = current - lastCheck;
        lastCheck = current;
        allowance += (double) timePassed / 1000 * ((double) rate / per);
        if (allowance > rate) {
            allowance = rate;
        }
        if (allowance < 1) {
            //discard message
            log.info("discard message:{}", GsonUtils.toJson(cacheEvent));
        } else {
            eventPublisher.publishEvent(cacheEvent);//发布事件
            log.info("publish message:{}", GsonUtils.toJson(cacheEvent));
            allowance -= 1;
        }
    }
}

class UpdateCacheEvent{
    private Long id;
}
```

简单的测试代码如下:

```java
@Test
public void  bar(){
    InvestService.RateLimiter rateLimiter = new RateLimiter(5, 1);//限流为每秒钟处理5个请求
    for (int i = 0; i < 100; i++) {
        UpdateCacheEvent cacheEvent = new UpdateCacheEvent();
        cacheEvent.setId((long) (10000 + i));
        rateLimiter.rateLimit(cacheEvent);
    }
}
```

参考:
<https://stackoverflow.com/questions/667508/whats-a-good-rate-limiting-algorithm>