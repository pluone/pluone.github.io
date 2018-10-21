---
layout: post
title:  "Java线程的状态转换"
date:   2018-08-12 21:00:00 +0800
categories: 多线程和并发
excerpt_separator: <!--more-->
---

本文主要是Java中线程状态及其线程状态之间的转化关系  
<!--more-->
问:什么情况下线程的状态是TIMED_WAIT?  

```java
Thread.sleep(sleep_time);
o.wait(timeout);
t.join(timeout);
LockSupport.parkNanos(timeout);
LockSupport.parkUnitl(timeout);
```

![Java线程状态转换图](/assets/thread-state.png)
参考:  
<https://www.uml-diagrams.org/examples/java-6-thread-state-machine-diagram-example.html>