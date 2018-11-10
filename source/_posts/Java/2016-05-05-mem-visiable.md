---
layout: post
title: JVM内存可见性之use Or read-load-use
categories: Java
description:
keywords: jvm 内存可见性
date: 2016-05-05
---

> 问题描述：
如果两个线程都对静态共享变量shareVar有引用，其中一个线程1使用 shareVar，另外一个线程2对shareVar进行修改，那么线程1是否能够理解拿到修改后的值呢？

![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/85bf0b2656136362bbe81ace0ae8c981.png?zoom=2)

代码描述：


```java
public class StaticVarVisibility {

    public static boolean shareVar = true;

    public static void main(String[] args) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                int i = 0;
                //JVM优化导致，当频繁使用主存变量的时候只做use，并未做read-load-use
                while (shareVar) {
                    i++;
                    //循环1000次，进行new Object();
                    //synchronized (this) { } 会强制刷新主内存的变量值到线程栈?我认为不是，是因为同步操作是一个耗时操作，所以对is的使用从单纯的use变为read-load-use
                    //System.out.println(i); //println 是synchronized 的
                    //try {
                    //sleep操作释放了CPU，所以遵循JVM优化基准，尽可能保证工作内存和主存的及时同步，如果CPU一直被占用，就无法及时做到数据同步
                    //TimeUnit.MICROSECONDS.sleep(1);
                    //} catch (InterruptedException e) {
                    //}
                }
            }
        }).start();

        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
        }

        new Thread(new Runnable() {
            @Override
            public void run() {
                shareVar = false;  //设置shareVar为false，使上面的线程结束while循环
            }
        }).start();
    }
}
```

#### 程序现象：
如果线程1中循环体内只有i++，此时循环并不会立即停止下来，为什么呢？线程2已经对shareVar设置了false，为什么线程1没有及时获取到修改后的变量呢？

在线程1中无论是循环创建对象还是使用synchronized还是sleep，都会触发线程1对shareVar变量的可见性。
#### 结果分析：
为什么static的变量能够实现可见性？其实跟static是否无关，即使只是一个简单的变量，线程1也是可以读到修改后的数据的，只是时间不可预期而已。

那为什么开始循环没有及时停止呢？原因是在JVM的优化策略下，当线程1频繁使用主存变量shareVar的时候只做use，并未做read-load-use，所以变更后的数据未能及时同步，当执行创建对象、sleep、synchronized这些耗时操作时，cpu对共享变量shareVar的使用不再频繁，所以jvm就可以来保证数据的同步，并从单纯的use变为read-load-use。

当然在shareVar变量前主动加上volatile关键字，程序会立即停下来，因为这相当于告诉jvm当变量进行变更后强制进行read-load-use。
#### 总结：
  知晓JVM什么时候只进行use，什么时候进行read-load-use。 在后续的编程中，如果需要修改后的数据必须做到实时可见性，必须在前面添加volatile关键字，这可能在支付相关的业务中存在。

