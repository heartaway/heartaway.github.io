---
layout: post
title: 动态配置中心学习笔记
categories: Middleware
description:
keywords: diamond 动态配置
date: 2015-12-20
---

#### 客户端使用：
1. 首先需要绑定HOST：
xx.xx.xx.xx  diamond.configserver.net
前面的IP地址为Diamond-server的部署IP。
2. 应用Diamond-clien二方库并开始使用；

#### Diamond的核心原理：
主要范围4个部分：

1. Diamond-server的集群同步；
2. Client获取Server的地址；
3. Client从Server获取数据；
4. Client运行时感知Server的数据变化；

使用前必读的四篇文章：

* Diamond的核心原理：http://jm-blog.aliapp.com/?p=1592
* Diamond的架构：http://jm-blog.aliapp.com/?p=1606
* Diamond的容灾：http://jm-blog.aliapp.com/?p=1617
* Diamond与ZK的区别：http://jm-blog.aliapp.com/?p=2561

此时我们对Diamond大致上有了一定的了解，那Diamond到底是怎么实现的嗯？只有仔细研读源码才能了解的更清楚。

工程目录结构：

![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/92a78ada25e538858dbcc2482c0d92b2.png?zoom=2)

#### Server的实现：
1. 配置管理服务（ConfigService）：
    此服务用户配置信息的增删改查，内部为了了一个所有配置信息的ConcurrentHashMap，用于记录数据key和对应的数据的Md5值，由于只是记录数据的MD5值，所以在内存开销上也并不算大；
查询配置信息：是直接查询的数据库；
添加配置信息：先将配置信息保存到数据库中，然后更新缓存中的Md5值，再将配置信息保存到本地磁盘中，最后通知其他server节点更新配置信息；
更新配置信息：与保存配置信息流程基本类似；
删除配置信息：首先删除本地磁盘数据，再删除环境中的key，然后删除数据库中的配置，最后通知其他server节点；
2. Server之间的通知服务（NotifyService）：
    此服务在启动时首先会从本地文件node.properties中将所有的server访问IP地址加载到Properties中，当调用notifyConfigInfoChange时，会遍历所有的节点信息逐个通过httpGet请求发送指定的dataId和Group的数据变更请求；当其他Server接收到Notify请求时，会调用ConfigService的loadConfigInfoToDisk方法，将数据库中的配置信息加载到磁盘中（包含cache中的Md5值的变更）；
备注：这里的疑惑点是，当server水平扩展的时候，如何动态配置server的节点数据？Server的列表是从另外一台Http Server中获取，并一段时间更新一次；当然Diamond的这个版本没实现这个功能，默认的实现是静态加载node的方式；
3. 磁盘操作服务（DiskService）：
    此服务就是将配置信息写入本地磁盘，唯一需要注意的点是，并发写的控制，这里使用的是ConcurrentHashMap的putIfAbsent方法，当正在写的时候将key设人map，写完后，remove掉。
4. 定时任务服务（TimerTaskService）：
    服务启动的时候就起了一个scheduledExecutorService，将DumpConfigInfoTask加入其中，使用配置中的间隔时间，默认是600秒（10分钟）；DumpConfigInfoTask做的事情就是分页查询数据库里面的配置信息，然后批量更新到本地磁盘文件中，并修改ConfigService缓存中的数据Md5值；
##### 总结：
从server的实现上，我们可以学习到以下几点


* 使用MD5来比较文件内容是否发生变化；
* 多服务节点之间怎么做数据同步；
* 使用乐观锁来做文件的并发写操作；

#### Client的实现：
1. 监听器：
    * DefaultSubscriberListener 为默认的业务监听器集合，管理了所有注册进来的DataId监听器，当DefaultSubscriberListener收到配置信息时，会从所有dataId的监听器中找到关心此diataid的监听器集（Map的Key），然后逐个做异步的事件通知；ManagerListener 是针对一个dataId的配置信息进行监听的监听器接口，需要用户自己实现接收到数据的处理方式；
2. 订阅者：
    * 通过工程类DiamondClientFactory实现单例订阅者DefaultDiamondSubscriber，DefaultDiamondSubscriber 管理业务监听器聚集、Diamond的基本配置、本地配置处理器、Snapshot配置处理器、服务地址处理器等；DefaultDiamondSubscriber 实现了DiamondClientSub，拥有启动和终止’定时获取配置信息‘的方法：start()、stop(); start()方法中会间隔一段时间轮询一次配置信息的变更；
    * 轮询配置信息的变更分为三部分：
        1. 检查本地文件的配置信息LocalConfigInfoProcessor；
        2. 检查远程DiamondServer上的配置信息变更，DiamondServer那么多配置信息，如何知道那些更新了呢？原因很简单，远程DiamondServer上配置信息多，但是应用里面使用的配置信息是一定的，找出缓存里面的所有配置信息，然后排除掉那些指定使用本地文件配置信息的DataId后把所有的DataId:Group:Md5拼接成一个文本（每行一条数据），通过http请求DiamondServer，查看这批配置里面那些dataId的Md5发生了变更，对发生了变更的dataId等信息拼接起来作为响应内容，客户端拿到这些变化了的dataid后，逐条向DiamondServer请求具体的内容数据，然后将数据发送给监听了此dataId的监听器，并保存数据到SNAPSHOT中；
        3. 检查SNAPSHOT中配置文件；对没有获取本地配置，也没有成功从diamond server获取到配置的DataId，加载上一次的SNAPSHOT信息；
3. 数据处理器：
    * Diamond提供了三种数据处理器，本地配置文件处理器LocalConfigInfoProcessor（配置文件存放于~/diamond/data/config-data/组名/dataId名）、Diamond远程服务配置文件处理器ServerAddressProcessor（从DiamondServer上获取配置信息）、快照配置文件处理器SnapshotConfigInfoProcessor（配置文件存放于~/diamond/snapshot/组名/dataId名）
    * 当我们在~/diamond/data/config-data下放置了配置信息时，优先使用本地配置文件（如果存在此配置，则此配置会被标识为使用本地），如果本地文件与内存中的配置有差异时会触发内存数据更新，即popConfigInfo给此dataId的监听器发信息；如果没差异则跳过；被标识使用本地配置的dataId，在进行远程DiamondServer配置检查的时候，是被跳过的；
    * 无论使用本地配置还是远程配置，都会将配置信息保存到本地快照中（popConfigInfo中实现），用户当Diamond不可用的时候，可以从SNAPSHOT中进行加载；
    * 从实现上我们了解到Diamond的配置信息是异步获取到的，那我们在使用时，如何保证第一次读取一定能拿到配置信息呢？
    * 原因：自己在实现ManagerListener的时候，构造函数中可以使用默认的DefaultDiamondManager，然后主动调用defaultDiamondManager.getAvailableConfigureInfomation(5000)来获取配置信息；
    * DefaultDiamondManager中getConfigureInfomation 和 getAvailableConfigureInfomation 方法的区别：getConfigureInfomation只从本地和远程DiamondServer上获取配置信息；
getAvailableConfigureInfomation 首先会从本地和远程DiamondServer上获取配置信息，如果都没有则从SNAPSHOT中加载配置信息；

##### 总结：
从Client的实现上，我们学习到以下几点

* 如何做多级容灾策略（强制指定配置、从DiamondServer上拉取配置 或 使用SNAPSHOT中的配置）；
* 订阅者模式的应用；
* 接口设计上的巧妙，给用户留了很多可以自由选择的地方，比如接收到数据内容后的格式处理、初始化获取配置的方式等；
* 合理的使用cache；

