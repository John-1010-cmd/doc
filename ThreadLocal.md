---
title: ThreadLocal
date: 2023-05-04
updated : 2023-05-04
categories: 
- Java
tags: 
- ThreadLocal
description: 这是一篇关于ThreadLocal的Blog。
---

## ThreadLocal简介

ThreadLocal叫做线程变量，意思是ThreadLocal中填充的变量属于当前线程，该变量对其他线程而言是隔离的，也就是说该变量是当前线程独有的变量。ThreadLocal为变量在每个线程中都创建了一个副本，那么每个线程可以访问自己内部的副本变量。

ThreadLoal 变量，线程局部变量，同一个 ThreadLocal 所包含的对象，在不同的 Thread 中有不同的副本。这里有几点需要注意：

因为每个 Thread 内有自己的实例副本，且该副本只能由当前 Thread 使用。这是也是 ThreadLocal 命名的由来。
既然每个 Thread 有自己的实例副本，且其它 Thread 不可访问，那就不存在多线程间共享的问题。
ThreadLocal 提供了线程本地的实例。它与普通变量的区别在于，每个使用该变量的线程都会初始化一个完全独立的实例副本。ThreadLocal 变量通常被private static修饰。当一个线程结束时，它所使用的所有 ThreadLocal 相对的实例副本都可被回收。

总的来说，ThreadLocal 适用于每个线程需要自己独立的实例且该实例需要在多个方法中被使用，也即变量在线程间隔离而在方法或类间共享的场景。

## ThreadLocal与Synchronized的区别

ThreadLocal其实是与线程绑定的一个变量。ThreadLocal和Synchonized都用于解决多线程并发访问。

但是ThreadLocal与synchronized有本质的区别：

1. Synchronized用于线程间的数据共享，而ThreadLocal则用于线程间的数据隔离
2. Synchronized是利用锁的机制，使变量或代码块在某一时该只能被一个线程访问。而ThreadLocal为每一个线程都提供了变量的副本，使得每个线程在某一时间访问到的并不是同一个对象，这样就隔离了多个线程对数据的数据共享。

一句话理解ThreadLocal，threadlocal是作为当前线程中属性ThreadLocalMap集合中的某一个Entry的key值Entry（threadlocl,value），虽然不同的线程之间threadlocal这个key值是一样，但是不同的线程所拥有的ThreadLocalMap是独一无二的，也就是不同的线程间同一个ThreadLocal（key）对应存储的值(value)不一样，从而到达了线程间变量隔离的目的，但是在同一个线程中这个value变量地址是一样的。

## ThreadLocal的简单使用

```java
public class ThreadLocaDemo {
 
    private static ThreadLocal<String> localVar = new ThreadLocal<String>();
 
    static void print(String str) {
        //打印当前线程中本地内存中本地变量的值
        System.out.println(str + " :" + localVar.get());
        //清除本地内存中的本地变量
        localVar.remove();
    }
    public static void main(String[] args) throws InterruptedException {
 
        new Thread(new Runnable() {
            public void run() {
                ThreadLocaDemo.localVar.set("local_A");
                print("A");
                //打印本地变量
                System.out.println("after remove : " + localVar.get());
               
            }
        },"A").start();
 
        Thread.sleep(1000);
 
        new Thread(new Runnable() {
            public void run() {
                ThreadLocaDemo.localVar.set("local_B");
                print("B");
                System.out.println("after remove : " + localVar.get());
              
            }
        },"B").start();
    }
}
//输出
//A :local_A
//after remove : null
//B :local_B
//after remove : null
```

## ThreadLocal常见使用场景

ThreadLocal 适用于如下两种场景

- 每个线程需要有自己单独的实例
- 实例需要在多个方法中共享，但不希望被多线程共享

**场景**

1. 存储用户Session
2. 数据库连接，处理数据库事务
3. 数据跨层传递（controller,service, dao）
4. Spring使用ThreadLocal解决线程安全问题 

## ThreadLocal内存泄漏原因

Entry将ThreadLocal作为Key，值作为value保存，它继承自WeakReference，注意构造函数里的第一行代码super(k)，这意味着ThreadLocal对象是一个“弱引用”。

主要两个原因

1. 没有手动删除这个 Entry
2. CurrentThread 当前线程依然运行

**由于ThreadLocalMap 的生命周期跟 Thread 一样长，对于重复利用的线程来说，如果没有手动删除（remove()方法）对应 key 就会导致entry(null，value)的对象越来越多，从而导致内存泄漏．**

### 为什么不将key设置为强引用

#### key 如果是强引用

如果key设计成强引用且没有手动remove()，那么key会和value一样伴随线程的整个生命周期。

假设在业务代码中使用完ThreadLocal, ThreadLocal ref被回收了，但是因为threadLocalMap的Entry强引用了threadLocal(key就是threadLocal), 造成ThreadLocal无法被回收。在没有手动删除Entry以及CurrentThread(当前线程)依然运行的前提下, 始终有强引用链CurrentThread Ref → CurrentThread →Map(ThreadLocalMap)-> entry, Entry就不会被回收( Entry中包括了ThreadLocal实例和value), 导致Entry内存泄漏也就是说: ThreadLocalMap中的key使用了强引用, 是无法完全避免内存泄漏的。

#### 为什么 key 要用弱引用

事实上，在 ThreadLocalMap 中的set/getEntry 方法中，会对 key 为 null（也即是 ThreadLocal 为 null ）进行判断，如果为 null 的话，那么会把 value 置为 null 的．这就意味着使用threadLocal , CurrentThread 依然运行的前提下．就算忘记调用 remove 方法，弱引用比强引用可以多一层保障：弱引用的 ThreadLocal 会被回收．对应value在下一次 ThreadLocaI 调用 get()/set()/remove() 中的任一方法的时候会被清除，从而避免内存泄漏。

##  如何正确的使用ThreadLocal

1. 将ThreadLocal变量定义成private static的，这样的话ThreadLocal的生命周期就更长，由于一直存在ThreadLocal的强引用，所以ThreadLocal也就不会被回收，也就能保证任何时候都能根据ThreadLocal的弱引用访问到Entry的value值，然后remove它，防止内存泄露。
2. 每次使用完ThreadLocal，都调用它的remove()方法，清除数据。