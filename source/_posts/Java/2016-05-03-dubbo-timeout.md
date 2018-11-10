---
layout: post
title: 记一次系统Dubbo调用超时的故障
categories: Java
description:
keywords: dubbo timeout 超时
date: 2016-05-03
---

>现象：生产环境用户无法使用下单，订单无法交易。

异常日志：


```java
Caused by: com.alibaba.dubbo.remoting.TimeoutException: Waiting server-side response timeout. 
start time: 2016-05-03 15:45:29.514, end time: 2016-05-03 15:45:35.771, client elapsed: 0 ms, 
server elapsed: 6257 ms, timeout: 1000 ms, request: Request [id=288854, version=2.0.0, twoway
=true, event=false, broken=false, data=RpcInvocation [methodName=getItemInfoListForOrder, par
ameterTypes=[interface java.util.List], arguments=[[xxx, xxx, xxx, xxx, xxx, xxx, xxx,
 xxx, xxx, xxx]], attachments={id=xxx, input=xxx, path=
```

## 分析:

发现订单调用商品的API超时了，登陆商品系统并没有发现任何的异常调用，感觉订单的系统调用并没有抵达商品系统，后来陆续发现订单访问其他系统的Dubbo调用都超时了，由此可断定可能是订单系统的问题。首先想到的是数据库的链接数，查看RDS的连接数：

![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/2583a7eee58bc603641dd69762c072f5.png?zoom=2)

可以看到，15点开始，总连接数开始飙升，并且临近最大值480(但是一直没到最大值480)，但是活跃连接数却始终保持在非常低的水平。可能是由于活动开始，根据之前经验，连接数上升是正常的情况，活动连接数低说明db在获取数据中，并且db的其他性能并没有出现瓶颈情况，iops情况如下：

![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/35f8ad8500739b5ff44ea28f054c2db0.png?zoom=2)

看了一下rds的性能优化发现有一条慢sql，但此时重点并没有放在这里。原因如上，我们认为db没有到达瓶颈（连接池没满，而且iops在正常范围）。

当时我们还查看了db的实时会话，当时有大量的处于sleep中的语句，大约有200多个,这说明，有大量sql连接正在等待mysql返回数据，这就是iops不高的原因。上图这是事后正常的截图。

另外一边进行jstack， 查看日志，发现大量线程处于wait状态：


```shell
[admin@order2.hz /home/admin]$grep 'java.lang.Thread.State:' 17116.jstack |
awk '{FS=":"} {print $2}'|awk '{print $1}' |sort |uniq -c
     18 RUNNABLE
    147 TIMED_WAITING
    662 WAITING
```

初步判断应该不竞争锁，否则出现的应该是blocked， Locked ownable synchronizers: – None 也说明了这个情况，也没有找到死锁的迹象。
我们查看Mysql的使用情况，发现200多个实物处于Sleep状态，但是没能推测出是由于大数据量查询都处于等待数据的返回。
接下来查看日志里面有**java.lang.OutOfMemoryError: Java heap space **，查看GC的情况，发现系统每5s就进行了一次FullGC，此时在查看内存情况，堆内存基本被占满。
继续查看日志，

![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/b8986c22adbd6ebf3296e803f04a32de.png?zoom=2)

![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/bd6226fb951d4dbd1cdb5b0cee0416b8.png?zoom=2)

发现由于一个SQL的query操作引起的，在跟进这段代码深入分析，发现这段查询逻辑直接查询数据库，而且数据量非常大，每次3W条数据，大于每个线程需要占用30M的堆内存，100个连接就需要3G的内存，但是我们的JVM最大设置的是2G内存所以导致存储空间不足。这下明白了，由于堆内存空间不足，所以任何dubbo的调用都无法正常分配到内存，所以就导致了任何接口的超时表现。

异常日志中出现 Unable to create new native thread，出现这个异常的原因、排查方法是什么呢？根据毕玄的文档：

![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/fa8b9a68b8d1d2e556950b05f9f9d5a3.png?zoom=2)

![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/af92db0be6556aae1394fbc6433dd842.png?zoom=2)

查看操作系统Linux的最大线程数：


```shell
[admin@order1.hz /home/admin]$cat /proc/sys/kernel/threads-max
```

结果是：60949 。
ulimit 用于限制 shell 启动进程所占用的资源，支持以下各种类型的限制：所创建的内核文件的大小、进程数据块的大小、Shell 进程创建文件的大小、内存锁住的大小、常驻内存集的大小、打开文件描述符的数量、分配堆栈的最大大小、CPU 时间、单个用户的最大线程数、Shell 进程所能使用的最大虚拟内存。同时，它支持硬资源和软资源的限制。


```shell
[admin@order1.hz /home/admin]$ulimit -a
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 30474
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files                      (-n) 65535
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 10240
cpu time               (seconds, -t) unlimited
max user processes              (-u) 1024
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited
```

max user processes Linux默认限制用户最大的线程数为1024。那么在本系统中JVM中最大的进程数是多少呢？
Jvm（jdk7）中，-Xms2g -Xmx2g  未设置Xss（the stack size for each thread），影响JVM线程数的因子有堆大小(Xmx)、每个线程的栈大小(Xss)、系统最大线程数据数（ulimit -u）,参考[：JVM中可生成的最大Thread数量。](http://jzhihui.iteye.com/blog/1271122)

未设置Xss，默认值是1M（[参考1](http://hg.openjdk.java.net/jdk7u/jdk7u/hotspot/file/2cd3690f644c/src/os_cpu/linux_x86/vm/globals_linux_x86.hpp#l32) 、[参考2](http://hg.openjdk.java.net/jdk7u/jdk7u/hotspot/file/2cd3690f644c/src/os_cpu/linux_x86/vm/os_linux_x86.cpp#l654)），至于操作系统上栈大小（ulimit -s）为10M，这个配置只影响进程的初始线程；后续用pthread_create创建的线程都可以指定栈大小。HotSpot VM为了能精确控制Java线程的栈大小，特意不使用进程的初始线程（primordial thread）作为Java线程，相关参考：

[What the difference between -Xss and -XX:ThreadStackSize is?](http://mail.openjdk.java.net/pipermail/hotspot-dev/2011-June/004272.html)

[Inconsistency between -Xss and -XX:ThreadStackSize in the java launcher](http://mail.openjdk.java.net/pipermail/hotspot-dev/2011-July/004288.html)

能够创建线程的最大个数的估算公式：

>(MaxProcessMemory – JVMMemory – ReservedOsMemory) / (ThreadStackSize) = Number of threads

通过大致计算，xmx=2G、xss=1M，那么4G –  2G – 300M(根据top参数预估) / 1M = 1700  预计可以创建1700个左右的线程，但是受限于ulimit -u ：1024的限制，所以最多可以创建1024个进程。
查看Jvm进程当前的进程数：


```shell
ps -eLf |grep java -c
```

未做活动是的数量为800，如果不修改ulimit的值，这很容易导致线程数超限的问题，所以建议修改到2000。


## 总结：
整个问题定位花费了很长时间，起先没有能从订单的错误日志入手，其次，监控信息的缺失，FullGC、内存这么重要的监控指标没能监控起来。所以以后遇到这样的类似的问题应该第一时间查看日志，其次查看各种指标数据CPU、内存、GC、jstack等；其次，在编码上，一定要注意任何的查询是否可能会导致大量数据的返回，如果有这种设计就应该重新设计方案。

