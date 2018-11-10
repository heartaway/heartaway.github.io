---
layout: post
title: Task混用ThreadPool导致无限等待
categories: Java
description:
keywords: task threadpool
date: 2016-08-17
---
### 现象：
生产环境商品打标异步任务提交任务后，任务没有被执行；查看日志，没有异常日志抛出。
### 初步猜测：
可能是队列出现了饱和或者死锁，但是如果出现了饱和，我们设置的线程池设置的饱和策略是通过主线程去执行，为什么主线程也没有执行呢？

### 具体分析：
我定义了一个线程池Pool-Z，core_size=5,max_size=20,queue_size=1000，第一个任务A提交后，占用一个线程，那么这个任务A又会被分解成多个异步任务去执行，比如分解为A1、A2、A3（这些子任务为耗时任务），这些异步任务使用了同一个线程池Pool-Z提供线程，但是任务A内部做了一个事情，通过CountDownLatch.await实现了当所有A的子任务都执行完毕后，才执行后续的一个扫尾工作，也就是说线程A会等待同一个线程池中的A1、A2、A3执行完毕后才能释放出线程A的资源；如果没有其它的主任务B或者C加进来，这些任务总会被执行完毕；但是如果主任务提交非常频繁，比如连续提交了几个包含非常多子任务的B、C、D进来，此时这样的不敢具体子任务只是等待子任务完成的主任务就占用了4个主要线程，只有一个子任务在执行工作，由于不会分配新的线程来处理子任务，所以B、C、D的子任务都会被扔到队列里面，知道队列满掉，当队列慢了后，会开辟新的线程来处理任务，但是我们知道由于很多主任务提交进来都处于等待子任务释放而处于等待状态，如果当某一刻20个核心线程都成为了主任务且都在等子任务执行完毕的时候，子任务就得不到线程，锁等待就可能发生了，因为当时配置的任务队列比较大，为3000，所以当时加入的任务都被加入到了队列中，由于总数没有达到3000个，所以没有触发线程池的饱和策略。
### 解决方案：
* 方案一是去掉锁的使用，且需要保证相同业务含义；实现较困难；
* 方案二把两个线程池拆开，不共用队列，避免了相互等待而产生死循环；
### 总结：
* 尽量避免线程之间的依赖；处理“当子任务都完成后再执行主任务”的场景最容易产生这样的线程依赖问题；
* 尽量避免使用wait、lock等操作；如果必须使用也切记加入超时时间；
* 对线程池的使用时最好自定义线程池的名称，方便拍错和定位问题；
* 如果需要使用内部类且内部类不需要应用外部类的属性，尽可能把内部类设置为static形态；

### 实践：
自定义线程工厂实现给线程重命名的实践：
```java
public  class NamedThreadFactory implements ThreadFactory {
    private static final AtomicInteger poolNumber = new AtomicInteger(1);
    private final ThreadGroup group;
    private final AtomicInteger threadNumber = new AtomicInteger(1);
    private final String namePrefix;
    private final boolean threadDeamon;

    NamedThreadFactory(String name) {
        SecurityManager s = System.getSecurityManager();
        group = (s != null) ? s.getThreadGroup() : Thread.currentThread().getThreadGroup();
        String newname = name != null ? name + "-thread" : "thread";
        namePrefix = "pool-" + poolNumber.getAndIncrement() + "-" + newname + "-";
        threadDeamon = false;
    }

    NamedThreadFactory(String name, boolean deamon) {
        SecurityManager s = System.getSecurityManager();
        group = (s != null) ? s.getThreadGroup() : Thread.currentThread().getThreadGroup();
        String newname = name != null ? name : "thread";
        namePrefix = "pool-" + poolNumber.getAndIncrement() + "-" + newname + "-";
        threadDeamon = deamon;
    }

    @Override
    public Thread newThread(Runnable runnable) {
        Thread thread = new Thread(group, runnable, namePrefix + threadNumber.getAndIncrement(), 0);
        thread.setDaemon(threadDeamon);

        if (thread.getPriority() != Thread.NORM_PRIORITY) {
            thread.setPriority(Thread.NORM_PRIORITY);
        }

        thread.setUncaughtExceptionHandler(new Thread.UncaughtExceptionHandler() {
            @Override
            public void uncaughtException(Thread thread, Throwable ex) {
                LOGGER.error("tagTask.", thread.getName() + " thrown an exception : ", ex);
            }
        });

        return thread;
    }
}
```

