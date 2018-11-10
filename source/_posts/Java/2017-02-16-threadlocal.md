---
layout: post
title: ThreadLocal 原理探究
categories: Java
description:
keywords: threadlocal
date: 2017-02-16
---

## ThreadLocal 的作用：
This class provides thread-local variables. 提供了线程级别的本地变量。

## ThreadLocal的原理：
1. 每一个线程Thread内部维护了两个package private级别的变量：
    * ThreadLocal.ThreadLocalMap threadLocals
    * ThreadLocal.ThreadLocalMap inheritableThreadLocals
2. 每一个 ThreadLocal 对象有一个创建时生成唯一的 Id称为：**threadLocalHashCode**，访问一个 ThreadLocal 变量的值，就是用threadLocalHashCode去 本线程 的 ThreadLocalMap 中查找对应的值。
   
```java
 private Entry getEntry(ThreadLocal<?> key) {
     int i = key.threadLocalHashCode 
     & (table.length - 1);
     Entry e = table[i];
     if (e != null && e.get() == key)
         return e;
     else
         return getEntryAfterMiss(key, i, e);
 }
```

3. ThreadLocalMap 为一个静态内部类，包含了一个Entity数组（Entry[] table）来保持ThreadLocal的线程变量；

## ThreadLocalMap的原理
1. ThreadLocalMap是一个Hash Map，采用的是开放地址法而不是链表的方式解决冲突。
2. Entry 是 WeakReference 的子类，这样不再被使用的 ThreadLocal 可以被检查出来并清除掉。
3. 线程隔离的秘密，就在于ThreadLocalMap这个类。ThreadLocalMap是ThreadLocal类的一个静态内部类，它实现了键值对的设置和获取（对比Map对象来理解），每个线程中都有一个独立的ThreadLocalMap副本，它所存储的值，只能被当前线程读取和修改。ThreadLocal类通过操作每一个线程特有的ThreadLocalMap副本，从而实现了变量访问在不同线程中的隔离。因为每个线程的变量都是自己特有的，完全不会有并发错误。还有一点就是，ThreadLocalMap存储的键值对中的键是this对象指向的ThreadLocal对象，而值就是你所设置的对象了。
   
```java
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
       void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

## 神奇的 0x61c88647
待补充；

## 使用陷阱：
*  ThreadLocal和线程池同时使用出现变量污染；
    在使用线程池的过程中，由于线程是不断的回收和利用故ThreadLocal在服务器中也是被反复利用的，在使用中如果不进行清查操作，很容易导致变量污染。 

## 问题：
#### 疑惑1：
既然ThreadLocal只维护一个值，每一个Thread中都存储的是一个副本，为什么ThreadLocalMap需要时一个数组而不是object呢？

**原因：**

如果我们期望在一个Thread中存储多个ThreadLocal变量怎么办呢？不可能改造Thread，新增一个Object对象吧，所以为了更方便的在同一个Thread中加入多个ThreadLocal变量，才引入了数组结构，不同的ThreadLocal对象作为不同键，所以在线程的ThreadLocalMap对象中设置不同的值。

#### 疑惑2：
ThreadLocalMap为什么采用开放地址的方法，而不采用链表来防止hash冲突？

答：待补充


## 扩张阅读：
http://qifuguang.me/2015/09/02/[Java%E5%B9%B6%E5%8F%91%E5%8C%85%E5%AD%A6%E4%B9%A0%E4%B8%83]%E8%A7%A3%E5%AF%86ThreadLocal/

