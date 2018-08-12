---
layout: post
title:  "Java线程的状态转换"
date:   2018-08-12 21:00:00 +0800
categories: java
---

![Java线程状态转换图](/assets/thread-state.png)

问:什么情况下线程的状态是TIMED_WAIT
Thread.sleep(sleep_time);
o.wait(timeout);
t.join(timeout);
LockSupport.parkNanos(timeout);
LockSupport.parkUnitl(timeout);

参考:  
<https://www.uml-diagrams.org/examples/java-6-thread-state-machine-diagram-example.html>