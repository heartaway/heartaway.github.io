---
layout: post
title: Java 中的单例模式
categories: Java
description:
keywords: singleton 单例模式
date: 2015-06-02
---

#### 方案一：非延迟加载单例类


```java
public class Singleton {  
　　private Singleton(){}  
　　private static final Singleton instance = new Singleton();  
　　public static Singleton getInstance() {
　　　　return instance; 　　
　　}
}
```

#### 方案二：简单的同步延迟加载


```java
public class Singleton {   
　　private static Singleton instance = null;  
　　public static synchronized Singleton getInstance() {
　　　　if (instance == null)
　　　　　　instance ＝ new Singleton();
　　　　return instance; 　　
　　}   
}
```

#### 方案三：双重检查成例延迟加载

```java
public class Singleton {   
　　private static volatile Singleton instance = null;  
　　public static Singleton getInstance() {
　　　　if (instance == null) {
　　　　　　　　synchronized (Singleton.class) {
　　　　　　　　　　　　if (instance == null) {
　　　　　　　　　　　　　　　　instance ＝ new Singleton();
　　　　　　　　　　　　}
　　　　　　　　}
　　　　}
　　　　return instance; 　　
　　}   
}
```

为什么不将 synchronized 关键字加载静态方法上呢？原因是，如果加载方法上，每次获取都会加锁，效率降低了很多。如果加载内部为空的方法里面，则只是在初始化的时候加锁一次。在可能的情况下,一般尽量将要同步的代码最小化, 这样可以达到线程的阻塞最小化。

#### 方法四：类加载器延迟加载


```java
public class Singleton {   
　　private static class Holder {
　　  static final Singleton instance = new Singleton();
　　}  

　　public static Singleton getInstance() {
　　　　return Holder.instance; 　　
　　}   
}
```
推荐方式四，利用内部类初始化时才进行单例类的初始化，实现了延迟加载，而且防止了并发，且避免使用锁的开销。

