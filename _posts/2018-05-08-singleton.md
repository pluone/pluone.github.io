---
layout: post
title:  "单例模式"
date:   2018-05-08 21:00:00 +0800
categories: 设计模式
---

单例模式很简单，但是要写好不简单，一个主要的原因是要考虑并发场景下如何安全的创建单例  
基本的思路是

1. 隐藏单例类的构造函数,防止通过new的方式创建单例
2. 通过暴露静态方法来提供实例

根据单例创建的时机分为两类:  

1. 如果是在类加载的时候就创建(Eager Initialization)，称为饿汉式  
2. 或者是在类使用的时候再加载(Lazy Initialization)，称为懒汉式

```java
//饿汉式写法，最容易理解，实例在加载类的时候创建,不需要使用同步即可保证线程安全
public class Singleton{
    private static Singleton singleton = new Singleton();

    private Singleton{}

    public static Singleton getInstance(){
        return singleton;
    }
}
```

```java
//这是正常的懒汉式写法，使用了同步
public class Singleton{
    private static Singleton singleton;

    private Singleton{}
    public synchronized static Singleton getInstance(){
        if (singleton == null){
            singleton = new Singleton();
        }
        return singleton;
    }
}
```

```java
//懒汉式写法
//不需要使用同步(synchronzied)
//利用了jvm懒加载类的原理，通过定义嵌套类(nested class)实现了懒加载
public class Singleton{
    private Singleton(){}

    private static class Holder{
        private static Singleton singleton = new Singleton();
    }

    public static Singleton getInstance(){
        return Holder.singleton;
    }
}
```

还有一种不常用的方式，仅仅作为学习参考  
这种方式在java5以前比较流行，因为之前的synchronized方法性能较差，所以没有直接在方法上加synchronized关键字,而是使用双重检查(Double-checked Locking)的方式来进行懒加载.  

```java
//不推荐的写法
public class Singleton {
    private Singleton() {}
    //注意这里的volatile是必须的
    private static volatile Singleton instance;

    public static Singleton getInstance() {
        if (instance == null) {
            // 加锁
            synchronized (Singleton.class) {
                // 这一次判断也是必须的，不然会有并发问题
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

参考:  
《Java Concurrency In Practice》第16章  
[https://javadoop.com/post/design-pattern#toc4](https://javadoop.com/post/design-pattern#toc4)