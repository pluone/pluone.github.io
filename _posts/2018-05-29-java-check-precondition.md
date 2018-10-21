---
layout: post
title:  "Java开发中判断前置条件的几种方式"
date:   2018-05-29 15:00:00 +0800
categories: 后端开发
---

Java开发中常常需要检查一个方法的参数是否满足特定条件,满足才能进行下一步.
例如:

```java
public void login(String phone,String password){
    //需要首先判断传入的参数是否为空
    //一般的项目都是spring项目,所以可以直接借助Spring的Assert来判断
    Assert.notNull(phone,"手机号不能为空");
    Assert.notNull(password,"密码不能为空");
}
```

但是看了`org.springframework.util.Assert`的注释有一句是这么写的:
**Mainly for internal use within the framework.**
该方法主要供spring 框架内部使用,于是就想到有没有第三方的库可以来做这种工作,一番google之后有如下发现:

```java
//进行前置条件判断的五种方式如下:
public void foo(String name, int start, int end){
    //第一种方式通过纯粹的java代码进行前置条件的检查
    if (null == name) {
        throw new NullPointerException("Name must not be null");
    }

    if (start >= end) {
        throw new IllegalArgumentException("Start (" + start + ") must be smaller than end (" + end + ")");
    }

    //第二种方式,借助于Spring框架的Assert类
    Assert.notNull(name, "Name must not be null");
    Assert.isTrue(start < end, "Start (" + start + ") must be smaller than end (" + end + ")");

    //第三种方式,借助于Apache Commons Validate
    Validate.notNull(name, "Name must not be null");
    Validate.isTrue(start < end, "Start (%s) must be smaller than end (%s)", start, end);

    //第四种方式,借助于Google Guava库Preconditions
    Preconditions.checkNotNull(name, "Name must not be null");
    Preconditions.checkArgument(start < end, "Start (%s) must be smaller than end (%s)", start, end);

    //第五种方式,使用java语言的assert关键字
    assert null != name : "Name must not be null";
    assert start < end : "Start (" + start + ") must be smaller than end (" + end + ")";
}
```

这五种方式的比较推荐的是Guava Preconditions和Apache Commons Validate,可以根据项目情况进行选择

参考:

1. [Spring Assert Statements](http://www.baeldung.com/spring-assert)
2. [Comparison of Ways to Check Preconditions in Java - Guava vs. Apache Commons vs. Spring Framework vs. Plain Java Asserts](http://www.sw-engineering-candies.com/blog-1/comparison-of-ways-to-check-preconditions-in-java)