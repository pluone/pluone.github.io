---
layout: post
title:  "Semaphore(信号量)"
date:   2018-07-12 21:00:00 +0800
categories: 多线程和并发
---

信号量用于控制同一时刻可以同时访问某一资源的最大允许数量，常见的使用场景是连接池。  
拿数据库连接池作为使用场景，信号量的初始值代表了连接池的最大数量，外部线程从连接池获取一个连接，可用连接的数量就会减1;  
直到连接池可用连接数量为零时，外部线程被阻塞，此时如果有其它线程释放掉了占用的连接，该等待线程即可获得连接。

```java
Semaphore sem = new Semaphore(n);//n表示许可的最大数量
void acquire();//用于获取一个可用的许可
void release();//用于释放一个占有的许可
```

下面这个示例代码实现了一个有限容量的HashSet,往set中添加元素时需要获取信号量,删除元素时则释放信号量

```java
public class BoundedHashSet<T> {
    private final Set<T> set;
    private final Semaphore sem;

    public BoundedHashSet(int bound) {
        this.set = Collections.synchronizedSet(new HashSet<T>());
        sem = new Semaphore(bound);
    }

    public boolean add(T o) throws InterruptedException {
        sem.acquire();
        boolean wasAdded = false;
        try {
            wasAdded = set.add(o);
            return wasAdded;
        } finally {
            if (!wasAdded)
                sem.release();
        }
    }

    public boolean remove(Object o) {
        boolean wasRemoved = set.remove(o);
        if (wasRemoved)
            sem.release();
        return wasRemoved;
    }
}
```