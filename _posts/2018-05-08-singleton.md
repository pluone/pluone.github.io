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
    private Integer tmp;
    private Singleton() {
        tmp = 10;
    }
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

### 关于双重检查写法中instance属性必须用volatile修饰的解释

**前提**
首先假设去掉第8行代码的volatile修饰  
上述第13-18行是同步代码块，
第11，20两行代码没有同步措施。
根据happens-before规则，监视器解锁操作 happens-before 同监视器的加锁操作，如果一个操作 happens-before 于另一个操作，那么我们说第一个操作对于第二个操作是可见的，因此第13-18行代码内部是可以保证可见性的，但是第11，20两行是没有可见性保证的。

**第一种情况**
线程A执行完了13-18行的代码，并且释放了监视器锁，
此时线程B开始执行第11行代码，因为没有可见性保证，所以线程B可能看到instance等于null或者不等于null，存在这两种情况，执行到第20行的时候同样可能返回的是null或者不是null

**第二种情况**
线程A执行到了第16行代码，并且变量的初始化和赋值发生了指令重排序，造成instance已经被赋值，但是构造函数尚未执行完毕。
此时线程B开始执行第11行代码，同样因为没有可见性保证，线程B可能看到instance等于null或者不等于null, 不等于null的这种情况会更糟糕，因为此时构造函数还未执行完毕，第17行返回的将是一个部分创建的对象(partially constructed object)

参考:  
《Java Concurrency In Practice》第16章  
[https://javadoop.com/post/java-memory-model](https://javadoop.com/post/java-memory-model)  
[stackoverflow](https://stackoverflow.com/questions/7855700/why-is-volatile-used-in-double-checked-locking)