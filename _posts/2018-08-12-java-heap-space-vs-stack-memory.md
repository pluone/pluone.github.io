---
layout: post
title:  "JVM堆内存和栈内存的区别"
date:   2018-08-12 22:00:00 +0800
categories: jvm
---

很多人都知道,JVM的内存空间划分为新生代,老年代,永久代,其中新生代和老年代的空间统称为堆区(heap),如图

<img src="/assets/jvm-memory-model.png" alt="虚拟机内存模型" style="width: 400px;"/>

还有一个概念是虚拟机栈(JVM Stack),是由栈桢(Stack Frame)组成的,栈桢内部有局部变量表和操作数栈,虚拟机栈属于线程私有内存空间,线程有pc计数器表示当前线程执行的位置.  
这是运行时数据区的一张图片,显示了堆区和栈区:  
<img src="/assets/jvm-runtime-data-area.jpg" alt="jvm运行时数据区" style="width: 400px;"/>

但是虚拟机栈区和堆区之间的区别又是什么呢?
下面用一段示例程序阐明它们之间的关系:

```java
public class Memory {
    public static void main(String[] args) { // Line 1
        int i=1; // Line 2
        Object obj = new Object(); // Line 3
        Memory mem = new Memory(); // Line 4
        mem.foo(obj); // Line 5
    } // Line 9

    private void foo(Object param) { // Line 6
        String str = param.toString(); //// Line 7
        System.out.println(str);
    } // Line 8
}
```

下图表示这段代码中堆区和栈区内存分配  
<img src="/assets/Java-Heap-Stack-Memory.png" alt="堆栈内存分配" style="width: 550px;"/>

1. 程序从main方法开始执行,虚拟机创建main方法的栈桢
2. 之后执行第二行代码i=1这个变量是放在栈桢的局部变量表中的
3. 之后执行第3行和第4行代码为对象分配内存空间,这个内存空间是在堆区进行分配的,栈桢中的局部变量表只是存储内存的引用地址
4. 之后第5行代码调用foo()此时虚拟机为foo方法创建一个新的栈桢,foo方法执行完毕后栈桢的空间被释放

下面划重点,堆内存和栈内存的主要区别:

1. 堆内存是所有线程共享的内存空间,而栈内存是线程私有的内存空间
2. 对象的内存空间分配总是发生在堆区,栈内存中则包含了对象的引用地址
3. 堆内存满了的时候抛出java.lang.OutOfMemoryError: Java Heap Space,而栈内存满了的时候抛出java.lang.StackOverFlowError
4. 栈内存相比堆内存小到微不足道.

参考:  
<https://www.journaldev.com/4098/java-heap-space-vs-stack-memory>