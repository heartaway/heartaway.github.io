---
layout: post
title: Java Load CPU MEM 飙高的一次排查
categories: Java
description:
keywords: jstack
date: 2017-06-15
---
### 问题发现：
收到报警提醒：
![](/images/posts/20170615/sys-mail-notice.png)


### 问题排查：

登录机器查看：
![](/images/posts/20170615/sys-load-top.png)

CPU 持续高于 80%，8G内存几乎被占满；

异常报警项为 一个每隔5min执行一次的定时任务每次运行时间超过10分钟；初步判断为任务累加导致系统负载越来越重。

那为什么定时任务在5min内执行未完成呢？看看线程到底在做什么。

查看进程中线程的占用CPU时长：
```java
top -Hp pid 
```
![](/images/posts/20170615/top-pid.png)

可以看到有不少进程占用CPU总计时间超过100分钟，最大的进程257484为564分钟32.32秒；
![](/images/posts/20170615/jstack-top.png)

查询其它的几个占用时间比较高的进程：
![](/images/posts/20170615/jstack-grep-tid.png)
发现都是ChangeFree的进程导致，查看相应代码后，发现最近的一次变更导致了程序在一些场景下出现了死循环：
![](/images/posts/20170615/changefree-while-query.png)
从代码可以看出，在while循环中的累加程序放在了最后，这是一种极危险的操作，应该讲累加条件放在程序进入的第一行，保证不受后续代码逻辑的影响；

问题解决后，通过系统监控发现系统负载恢复正常：
![](/images/posts/20170615/system-monitor-line.png)

