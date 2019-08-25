---
layout: post
title:  "策略模式"
date:   2019-07-06 16:00:00 +0800
categories: 设计模式
excerpt_separator: <!--more-->
---

策略模式(Strategy Pattern)：定义一系列算法，将每一个算法封装起来，并让它们可以相互替换。策略模式让算法独立于使用它的客户而变化。
> Define a family of algorithms, encapsulate each one, and make them interchangeable. Strategy lets the algorithm vary independently from clients that use it.

<!--more-->
## 策略模式的参与者

Strategy: 抽象策略类
ConcreteStrategy: 具体策略类
Context: 上下文

## 实际使用中的例子
公司内部的结算系统对接外部的结算渠道, 每个渠道都有自己的内部编码, 内部系统调用结算系统时提供渠道编码, 结算平台根据不同的编码请求不同的外部渠道.
但是外部渠道比较大, 大概有20多个, 而且还会不断的新增. 目前的伪代码如下:
```java
public void deduct(String channelCode){
    if("01".equals(channelCode)){
        requestChannel_1();
    }else if("02".equals(channelCode)){
        reqeustChannel_2();
    }else if("03".equals(channelCode)){
        reqeustChannel_3();    
    }else if("04".equals(channelCode)){
        reqeustChannel_4();
    }else if(){
        // ... 省略中间部分代码
    }else if("19".equals(channelCode)){
        reqeustChannel_19();
    }else if("20".equals(channelCode)){
        reqeustChannel_20();
    }

}
```

针对这样的情况可以使用策略模式进行优化, 可以将每一个请求渠道的请求抽象出一个接口
```java
// 划扣策略接口
public class DeductStrategy{
    // 进行资金划扣
    Object deduct();
}
```
这样不同的资金渠道可以实现这个接口
```java
// 宝付划扣渠道
public class BaoFooDeductStrategy implements DeductStrategy{
    @Override
    public Object deduct(){
        // 封装具体的http请求
    }
}

// 京东划扣渠道
public class JDDeductStrategy implements DeductStrategy{
    @Override
    public Object deduct(){
        // 封装具体的http请求
    }
}
```

调用的代码则变化为
```java
public class Deduct{
    public void deduct(String channelCode){
        // 通过spring的容器工厂获取到具体的实现类, 进行调用
        DeductStrategy deductStrategy = ContextLoader.getCurrentWebApplicationContext().
                    getBean(DeductChannelEnum.getClazzByChannelCode(channelCode));
        deductStrategy.deduct();
    }
}

// 定义枚举类
public enum DeductChannelEnum{
    BaoFoo("02", BaoFooDeductStrategy.class),
    JD("03", JDDeductStrategy.class);
    private String channelCode;
    private Class clazz;

    public static Class getClazzByChannelCode(String channelCode){
        // ...
    }
}
```

这样写的好处  
策略模式的实线类是单一职责的, 每个实线类只负责具体三方渠道的请求逻辑.  
如果后面继续新增渠道, 则只需添加具体的渠道实现类, 从而符合了开笔原则, 即对扩展开放, 对修改关闭.