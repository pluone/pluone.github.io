---
layout: post
title:  "CountDownLatch(倒计数锁存器)"
date:   2018-08-10 22:00:00 +0800
categories: concurrency
---

题外话:这片文章是5月5号写的,今天面试的时候提到这个场景,我竟然脑海中有却没敢答出来,还是知识掌握不够牢固啊.现在发出来温故知新一下.

Latch是门闩的意思，先看一下门闩的图片,联想一下为什么叫“倒计时门闩”,门闩缓缓移动，只有到从门搭上离开的那一刻门才能打开。  
![Latch](/assets/latch.png)

在多线程环境中只有当门闩打开，所有的线程才能继续运行，否则就要一直等待门闩打开。

CountDownLatch是这样一种机制，初始化的时候传入一个整数，整数值表示要等待多少个事件发生，每发生一个事件countDown()方法自动将整数值减1  
直到所有事件全部发生,所有的等待线程即可继续运行。

```java
CountDownLatch startGate = new CountDownLatch(1);//初始化整数值，表示要等待的事件个数
startGate.await();//等待所有事件发生
startGate.countDown();//事件发生后，计数器的值减1
```

下面的示例代码用来测试n个线程同时开始运行到最后一个运行完毕所用的时间。  
其中定义了startGate和endGate两个倒计时锁存器,startGate的作用是在n个线程同时开启后等待计时开始口令,
endGate的作用是等待直到最后一个线程运行完毕。

```java
public class TestHarness {
    public long timeTasks(int nThreads, final Runnable task) throws InterruptedException {
        final CountDownLatch startGate = new CountDownLatch(1);
        final CountDownLatch endGate = new CountDownLatch(nThreads);
        for (int i = 0; i < nThreads; i++) {
            Thread t = new Thread() {
                public void run() {
                    try {
                        startGate.await();
                        try {
                            task.run();
                        } finally {
                            endGate.countDown();
                        }
                    } catch (InterruptedException ignored) {
                    }
                }
            };
            t.start();
        }
        long start = System.nanoTime();
        startGate.countDown();
        endGate.await();
        long end = System.nanoTime();
        return end - start;
    }
}
```