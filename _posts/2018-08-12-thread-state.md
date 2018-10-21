---
layout: post
title:  "Java线程的状态转换"
date:   2018-08-12 21:00:00 +0800
categories: 多线程和并发
excerpt_separator: <!--more-->
---

本文主要整理了Java中线程状态及其之间的转化关系  
<!--more-->
Thread类中一共定义了6中线程状态，如下：

```java
// JDK Thread类源码,Thread类中的无关代码省略
public class Thread{
    //...
    public static enum State {
        NEW,
        RUNNABLE,
        BLOCKED,
        WAITING,
        TIMED_WAITING,
        TERMINATED;
    }
}
```

这六种状态的含义

- NEW 表示线程处于新建状态，还未调用start()方法启动线程
- RUNNABLE 表示线程处于可运行状态
- BLOCKED 表示线程处于等待获取锁的状态
- WAITING 表示线程处于等待状态.  
  object.wait()方法,thread.join()方法,LockSupport.park()方法可使线程处于该种状态.
- TIMED_WAITING 表示线程处于有限时间的等待中.  
  thread.sleep(sleep_time),object.wait(timeout),object.join(timeout),LockSupport.parkNanos(timeout),LockSupport.parkUtil(time)可以使线程处于该状态.

线程状态转换图如下:  
![Java线程状态转换图](/assets/thread-state.png)
参考:  
<https://www.uml-diagrams.org/examples/java-6-thread-state-machine-diagram-example.html>