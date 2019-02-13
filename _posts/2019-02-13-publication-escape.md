---
layout: post
title:  "对象的发布和逸出"
date:   2019-02-13 20:45:00 +0800
categories: 多线程和并发
excerpt_separator: <!--more-->
---

本文主要总结了对象的发布和逸出，以及如何安全的发布对象。主要内容均由《Java Concurrency In Practice》第三章整理而来。
<!--more-->

## 发布(Publication)

发布对象是指，使对象能够在当前作用域之外的代码中使用，例如，将一个指向该对象的引用保存到其他代码可以访问的地方，或者在一个非私有的方法中返回该引用，或者将引用传递到其他类的方法中。

> Publishing an object means making it available to code outside of its current scope, such as by storing a reference to it where other code can find it, returning it from a nonprivate method, or passing it to a method in another class.

## 逸出(Escape)

当一个不应该发布的对象被发布时称为逸出。
> An object that is published when it should not have been is said to have escaped.

### this逸出(this escape)

this escape作为逸出的一种典型情况。

```java
public class ThisEscape {
    public ThisEscape(EventSource source) {
        source.registerListener(
            new EventListener() {
                public void onEvent(Event e) {
                    doSomething(e);
                }
            });
    }
}
```

这段代码在发布EventListener对象的同时隐含的发布了ThisEscape对象(这是因为new EventListener作为一个内部类隐含了外部对象的this指针)，但是ThisEscape的构造函数尚未执行完毕，因此发生了this逸出。

## 安全发布(safe publish)

### 不安全的发布

```java
// Unsafe publication
public Holder holder;
public void initialize() {
    holder = new Holder(42);
}

public class Holder {
    private int n;
    public Holder(int n) { this.n = n; }
    public void assertSanity() {
        if (n != n){
            throw new AssertionError("This statement is false.");
        }
    }
}
```

这段代码是一个不安全的发布的例子，Holder对象执行完inintialize()初始化代码后即被发布出去，其他线程可以直接访问到holder,但是因为存在内存可见性问题，holder对象对于其他线程可能处于一种不一致的状态。

不安全发布的对象可能存在以下两种问题：

1. 其他线程看到的值是过时的，holder对象可能是null或者更早之前的值。
2. 更严重的情况是其他对象可以看到最新的holder对象引用，但是holder对象内部状态n却是过时的。
一个线程可能初始看到的是一个过时的值，随后很快就变成了最新的值，这正是assertSanity方法可能会抛出AssertionError异常的原因。

### 如何安全发布

不可变对象可以被安全的访问，而不需要使用任何同步措施。
> Immutable objects can be used safely by any thread without additional synchronization, even when synchronization is not used to publish them.

可变对象必须被安全的发布，要安全的发布一个对象，对象的引用和对象的状态必须同时对其他线程可见。一个正确构造的对象可以通过以下方式来安全的发布：

- 在静态初始化函数中初始化一个对象引用
- 将对象的引用保存到volatile类型的域或者AtomicReference
- 将对象的引用保存到某个正确构造对象的final类型域中
- 将对象的引用保存到一个有锁保护的域中
  
> To publish an object safely, both the reference to the object and the object's state must be made visible to other threads at the same time. A properly constructed object can be safely published by:
>
- Initializing an object reference from a static initializer;
- Storing a reference to it into a volatile field or AtomicReference;
- Storing a reference to it into a final field of a properly constructed object; or
- Storing a reference to it into a field that is properly guarded by a lock.

举例如下：  
`public static Holder holder = new Holder(42);`
即为使用静态初始化器来初始化holder对象

往Hashtable,synchronizedMap,ConcurrentHashMap中放入key和value即完成了安全发布，任何其他的线程都可以安全的取出key和value，这是因为这些类的内部都使用了锁的保护。
