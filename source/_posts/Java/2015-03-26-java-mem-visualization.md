---
layout: post
title: Java 内存管理可视化
categories: Java
description:
keywords: jvm 内存管理 可视化
date: 2015-03-26
---
开源项目：[https://github.com/kittylyst/jfx-mem](https://github.com/kittylyst/jfx-mem) 提供给我们一个Java内存分配和GC的可视化过程。

![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/a2b296390844a629bc3a17c120dccb55.png)

#### 运行方法：

下载代码到本地，使用IDEA打开工程，新建配置，选择“Application”。

![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/1ea51c9fd021f3717bd4060c084e6809.png)

#### 区块解释：
eden区用户新分配内存区，也是使用最为频繁的区域，一般线程的TLAB（Thread Local Allocation Buffer）都是在eden去开辟，且线程间隔离。eden区 、survivor区、old区的内存之和为Heap区，其中eden区与servivor为Young Gen（新生代）。当eden区满后会触发一次YGC，eden区中还需要被使用的对象会被移到survivor区中（survivor一般为2个，也成为from区、to区），这样整个Eden区都是未被使用的空间，可共继续创建对象，当Eden区再次用完，再触发一次Yong GC，将Eden区和From区还在被使用的对象复制到To区，下一次Yong GC则是将Eden区和To区的还被使用的对象复制到From区。因此，进过多次Yong GC，某些对象会在From区和To区多次复制，如果超过某个阀值对象还未被释放，则将该对象复制到Old Generation。如果Old Generation空间也用完，那么就会触发一次Full GC，即所谓的全量回收，全量回收会对系统性能产生较大影响，因此应根据系统业务特点和对象周期，合理设置Yong Generation 和 Old Generation大小，尽量减少 Full GC。为什么Full GC会对系统性能产生较大影响，原因是Full GC不可避免的需要“stop the world” 让应用程序的所有经常暂停。

#### 执行过程：
从动画中我们还可以看出，Old区采用的跟踪计数器（相对于引用计数器）的算法是：标记-清除-压缩 的方式，那我们常用的跟踪计数器的算法有那些呢，各有什么优缺点？

#### 常见的垃圾回收方法有：
##### 1. 复制；
从根集合搜扫描出存活的对象，然后将存活的对象复制到一块新的未使用的空间中，当要回收的空间中存活的对象较少时，比较高效；
##### 2. 标记-清除；
从根集合开始扫描，对存活的对象进行标记，比较完毕后，再扫描整个空间中未标记的对象，然后进行回收，不需要对对象进行移动；弊端是造成内存碎片，可能内存有500M，当需要100M时就没有内存了，需要触发一次FGC。
##### 3. 标记-压缩；
标记形式和“标记清除”一样，但是回收不存活的对象后，会把所有存活的对象在内存空间中进行移动，好处是减少了内存碎片，缺点是成本比较高；

新时代中对象存活时间比较短，YGC次数也比较多，故可以采用效率比较高的复制策略，YGC触发时，把Eden区和From区还存在应用关系的复制到To区中，下一次是把Eden区和To区复制到From区。

新时代可用的GC方法和老年代可用的GC方法是不一样的：

* 新时代可用GC方法：串行GC（Serial GC）、并行回收GC（Parallel Scavenge）、并发GC（ParNew）
* 老年代可用GC方法：串行GC（Serial MSC）、并行GC（Parallel MSC）、并发GC（CMS）

GC 的组合类型（每个参数类型都对应了YGC、FGC采用的GC策略）：

| 参数 | 描述 |
| --- | --- |
|UseSerialGC	|虚拟机运行在Client模式的默认值，打开此开关参数后，使用Serial+Serial Old收集器组合进行垃圾收集。|
|UseParNewGC	|打开此开关参数后，使用ParNew+Serial Old收集器组合进行垃圾收集。|
|UseConcMarkSweepGC	|打开此开关参数后，使用ParNew+CMS+Serial Old收集器组合进行垃圾收集。Serial Old作为CMS收集器出现Concurrent Mode Failure的备用垃圾收集器。|
|UseParallelGC	|虚拟机运行在Server模式的默认值，打开此开关参数后，使用Parallel Scavenge+Serial Old收集器组合进行垃圾收集。|
|UseParallelOldGC|	打开此开关参数后，使用Parallel Scavenge+Parallel Old收集器组合进行垃圾收集。|

查看我们小二后台的jvm参数设置，我们使用的是CMS方式：

![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/5bb973bcf6eefcfd36fdedbc1a34a75e.png)

-XX:+UseConcMarkSweepGC 表示使用CMS策略；CMS策略默认只做标记-清除，并不做压缩，如果期望FGC的时候，同时做压缩，解决内存碎片的问题，可以采用的方式是添加参数：
>-XX:+UseCMSCompactAtFullCollection；

-XX:CMSInitiatingOccupancyFraction=80 表示只有在第一次Old区使用率超过80%时，自动触发CMS GC，后面都是使用HotSpot VM自动计算出来的值。

-XX:+UseCMSInitiatingOccupancyOnly 表示命令JVM不基于运行时收集的数据来启动CMS垃圾收集周期。而是，当该标志被开启时，JVM通过CMSInitiatingOccupancyFraction的值进行每一次CMS收集，而不仅仅是第一次。

-XX:+CMSClassUnloadingEnabled 相对于并行收集器，CMS收集器默认不会对永久代进行垃圾回收。如果希望对永久代进行垃圾回收，可用设置标志-XX:+CMSClassUnloadingEnabled。在早期JVM版本中，要求设置额外的标志-XX:+CMSPermGenSweepingEnabled。注意，即使没有设置这个标志，一旦永久代耗尽空间也会尝试进行垃圾回收，但是收集不会是并行的，而再一次进行Full GC。
如果想更深入了解CMS收集器的原理，可以参考：[http://ifeve.com/useful-jvm-flags-part-7-cms-collector/](http://ifeve.com/useful-jvm-flags-part-7-cms-collector/)

![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/aa638667a77acf42f9f8f2f9f96fe781.png)

从上图中可以看出，Old区满了，即将做一次FGC，如果FGC后内存还是满了，就会触发OutOfMemory。


