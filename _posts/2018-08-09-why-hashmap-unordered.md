---
layout: post
title:  "为什么说HashMap是无序的"
date:   2018-08-09 22:00:00 +0800
categories: 后端开发
---

HashMap和HashSet遍历元素时是无序的,这恐怕是一个常识了,但是你有没有想过为什么是无序的?
TreeMap和LinkedHashMap是有序的,那又为什么是有序的呢?

本节我们从源码分析的角度来理一理这件事情.  

首先说HashMap是如何遍历的,一般来说有两种遍历方式,我们用常用的这一种,使用entrySet的iterator方法,如下

```java
Map<String, Object> map = new HashMap<>();
Iterator<Map.Entry<String, Object>> iterator = map.entrySet().iterator();
while (iterator.hasNext()) {
    Map.Entry<String, Object> next = iterator.next();
    System.out.println(next.getKey() + "\t" + next.getValue());
}
```

```java
//源码来源于JDK1.8
public Set<Map.Entry<K,V>> entrySet() {
    Set<Map.Entry<K,V>> es;
    //我们看到这里new了一个EntrySet
    return (es = entrySet) == null ? (entrySet = new EntrySet()) : es;
}
```

```java
public final Iterator<Map.Entry<K,V>> iterator() {
    //然后我们看EntrySet的源码中的iterator返回了EntryIterator
    return new EntryIterator();
}

//这里是EntryIterator的定义,其中的next方法便是遍历元素的顺序
final class EntryIterator extends HashIterator
    implements Iterator<Map.Entry<K,V>> {
    //nextNode方法执行HashIterator中的nextNode(),
    public final Map.Entry<K,V> next() { return nextNode(); }
}

abstract class HashIterator {
    Node<K,V> next;        // next entry to return
    Node<K,V> current;     // current entry
    int expectedModCount;  // for fast-fail
    int index;             // current slot

    HashIterator() {
        expectedModCount = modCount;
        Node<K,V>[] t = table;
        current = next = null;
        index = 0;
        if (t != null && size > 0) { // advance to first entry
            do {} while (index < t.length && (next = t[index++]) == null);
        }
    }

    final Node<K,V> nextNode() {
        Node<K,V>[] t;
        Node<K,V> e = next;
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        if (e == null)
            throw new NoSuchElementException();
        //下面这个代码的意思就是如果数组当前位置为空就去寻找下一个位置,参考下图
        if ((next = (current = e).next) == null && (t = table) != null) {
            do {} while (index < t.length && (next = t[index++]) == null);
        }
        return e;
    }
    //部分源码省略
}
```

参考这张图片  
HashMap中元素的遍历是按照从数组起始位置开始,首先将当前bucket下的所有元素遍历完成,然后到下一个bucket,bucket与bucket之间如果为空就跳到下一个bucket,直到将所有的元素遍历出来.显而易见,元素插入的位置并不是这样的顺序,因此才说HashMap是无序的.
![sxx](/assets/hashmap.jpg)

## TreeMap和LinkedHashMap是如何实现有序的
TreeMap的底层数据结构是一棵红黑树,红黑树上的元素都是有顺序的.  
LinkedHashMap底层数据结构就是一个双向链表,元素遍历的顺序就是链表从前到后的顺序,因此也是有序的.