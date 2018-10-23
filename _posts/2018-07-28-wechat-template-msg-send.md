---
layout: post
title:  "RabbitMQ之微信模版消息发送时间超长问题"
date:   2018-07-28 18:00:00 +0800
categories: 后端开发
---

问题:通过mq给用户发微信通知,使用的是微信的模版消息,要求该消息具有较高的时效性,当前要给5000个用户发送消息,大概30分钟才能发送完毕,预期的要求是1-2分钟.

mq的配置为15个并发消费者,通过观察mq的图形化界面发现消费者消费的速度大约为0.5个每秒,这个速度绝对是有问题.

排查问题的思路如下:

1. 添加微信接口耗时统计后发现,微信接口的耗时竟然是递增的,类似于1000ms,1500ms,2000ms这样在递增,微信的接口不可能这么慢的,那一定是我们的代码哪里阻塞了
2. 利用排除法,将我们自己的业务逻辑隔离起来观察发现统计处理的耗时结果是一样的,说明问题还是出在了微信的接口调用上
3. 因为使用的接口是第三方提供的maven依赖,而且这个依赖的版本已经很老了,我们研究了这个库的源码,竟然有了惊人的发现

第三方依赖如下:

```xml
<dependency>
    <groupId>me.chanjar</groupId>
    <artifactId>weixin-java-mp</artifactId>
    <version>1.3.3</version>
</dependency>
```

依赖中发送模版消息的源码如下:

```java
  public String templateSend(WxMpTemplateMessage templateMessage) throws WxErrorException {
    String url = "https://api.weixin.qq.com/cgi-bin/message/template/send";
    //下面这一个执行的是发送http请求
    String responseContent = execute(new SimplePostRequestExecutor(), url, templateMessage.toJson());
    JsonElement tmpJsonElement = Streams.parse(new JsonReader(new StringReader(responseContent)));
    final JsonObject jsonObject = tmpJsonElement.getAsJsonObject();
    if (jsonObject.get("errcode").getAsInt() == 0)
      return jsonObject.get("msgid").getAsString();
    throw new WxErrorException(WxError.fromJson(responseContent));
  }
```

```java
  public <T, E> T execute(RequestExecutor<T, E> executor, String uri, E data) throws WxErrorException {
    int retryTimes = 0;
    do {
      try {
          //这里才是真正的执行http请求的地方
        return executeInternal(executor, uri, data);
      } catch (WxErrorException e) {
       ......//此处省略
    } while (++retryTimes < maxRetryTimes);

    throw new RuntimeException("微信服务端异常，超出重试次数");
  }
```

```java
//这个方法竟然加了synchronized锁,这个锁导致虽然有15个消费者,也就是15个工作线程,但是所有的线程都必须串行执行http请求
  protected synchronized <T, E> T executeInternal(RequestExecutor<T, E> executor, String uri, E data) throws WxErrorException {
   ......//代码省略
  }
```

起初我还以为是apache HttpClient的TCP连接池配置有什么问题呢,没想到竟然是这样,所有的线程都在串行执行.
找到了原因之后我们可以预估一下时间,(5000个用户*微信接口调用每次400ms/60)结果约等于33分钟,如果15个消费者可以并发消费那么速度可以提高15倍,时间就可以达到大约2分钟.

为了再次确认一下我们可以用jstack打出线程堆栈信息,确认一下其它的线程都在等待同步锁.(这里省略)

后记:因为这个maven依赖太老了,作者已经不维护了,转到了其它项目,我们采用了最新的依赖包,此问题在新的代码中已经得到了解决.