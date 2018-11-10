---
layout: post
title: 商品系统 之 Redis的使用
categories: Redis
description:
keywords: redis product 商品系统
date: 2016-06-29
---
 Redis在现在的电商系统中越来越多的被使用作为缓存体系，掌握基本的Redis的使用还是非常有必要的。

## Jedis 客户端初始化：
 你是如何初始化Jedis的？使用Spring Bean 或者 静态工具类？项目中有人使用在类的静态初始化块中初始化Jedis连接，有没有捕获异常，导致Jedis初始化失败后，整个类就无法进行加载，进而报ClassNotDefFound的错误，所以我们在初始化Jedis的时候是需要考虑异常场景的（当然使用原来外部资源的方式初始化静态变量就不是一个好的选择）。对于持久化Redis，当连接或请求异常的是，进行连接retry和reconnect，重试时间应该大于cluster-node-time时间，但是有一点需要考虑，当你的线程大量处于重试时，是否会达到线程池的最大值？

## Jedis业务代码中的容灾方案：

### ①请求容灾
在创业初期，一般公司都是缺乏专业的Redis运维人员，Redis的稳定性也得不到保证，所以这个时候就不能把Redis作为唯一的救命稻草，如果Redis挂了，你的业务也就跪了。在我们电商系统中，各个垂直系统之前公用Redis集群，后来发现如果一个节点出现问题就会导致整个公司的业务挂点，所以对Redis集群进行了业务拆分，不同的业务系统使用单独的Redis集群，持久化缓存和非持久化缓存进行分离。虽然这样在某种程度上降低了Redis宕机给业务带来的风险，但是如果在客户端这边没有对Redis的使用考虑容灾降级的话，也是可能给业务带来一连串的风险；比如，我们遇到了一次公共Redis集群中的几台ECS主机挂掉了，重启redis集群、重启ECS都不起作用，这个时候业务系统由于没有对Redis考虑容灾和降级，在获取Redis资源的时候就开始大量报错，导致使用了此服务的所有应用都出现了异常；如果Jedis客户端采用Spring Bean，由于拿不到Redis连接，还会让应用容器无法启动。这个时候如果通过容灾策略跳过查询Redis转为查询DB的话，大量请求直接打在DB上就可能导致DB瓶颈，但是如果Redis是集群的话，单个节点不可用只会影响落到这个Node的请求，当15s后，Node的Slave升为Master就可以解决缓存，给业务带了的不可用只是短暂的。

### ②hot-key 容灾
 避免产生hot-key，导致节点成为系统的短板；如果系统容易产生hot-key，建议系统进行本地缓存（比如使用ehcache），这种方案会对业务有侵入性，还有一种解决方案是搭建无状态的proxy，proxy具备缓存hot-key的能力，redis需要具备发现hot-key并进行主动上报proxy，这样hot-key就从单节点转移到了proxy层，同时proxy在压力大的场景下可以平滑水平扩容，当然你也需要考虑数据一致性问题。

### ③big-key 容灾
避免产生big-key，导致网卡打爆、慢查询；
### ④TTL
避免大量Key在同一时间段过期。最简单的办法是缓存时间随机，这对一次放入大量的Key是有效的，但是对于分散的key不起作用，而且随机不易管理，一个key到底什么时候失效我们无法做到管控。

### ⑤高并发key失效 容灾
在流量非常大的情况下，如果缓存失效，大量请求就会打到DB上，这个时候很可能达到DB的瓶颈，造成雪崩，所以为了有效防止这种场景下的雪崩，我们引入了 “分段式缓存策略”，将缓存分为缓存逻辑失效(expire)和缓存物理失效(maxExpire)；缓存物理失效时，查询数据重新刷入缓存，这点没什么变化；缓存逻辑失效后，当前线程会获取锁，并进行缓存更新，在缓存更新刷新的过程中，其它大量的请求可以直接返回老数据（只是逻辑过期，并未物理过期），从而有效防止雪崩。

分段式查询缓存:

1. 缓存未命中，尝试获取更新锁:
   
    * 获取更新锁成功，执行业务并刷新缓存;
    * 获取更新锁失败，说明缓存正在被更新，直接查询DB；
2. 缓存命中，缓存是否逻辑过期：
    * 缓存逻辑未过期，数据有效且正常，直接返回；
    * 缓存逻辑过期，获取更新锁：
        * 更新锁获取成功，执行业务刷新缓存；
        * 获取更新锁失败，说明有线程在刷新缓存直接返回逻辑过期的数据；

 **注意：** 当你进行缓存更新的之前需要获取锁，这个锁一般使用Redis实现分不是锁；获取到锁后，业务处理完毕记得释放锁。

## Redis操作中Key的定义：
随着业务的发展，Redis的使用越来越多，Redis的内存占用量也水涨船高，但是Redis的内存不是无限，有几种方案降低Redis机器内存占用的值:

①是增加更多的节点，拉平内存占用量水位；

②对数据进行压缩，解决内存碎片问题；

③缩减Key或者Value对Redis的使用量，比如简化Key的占用，或者对Key进行MD5。

④使用带压缩策略的Redis数据结构，比如Hash。


那么如何更优雅的定义Key呢 ？一般的做法是:

>系统名称前缀:表名称:字段名称:版本号:业务数据

设计版本号的作用是当新增字段时，需要更新缓存，只需要修改一下版本号就可以了。然后对这个规则的Key进行最大可能的字段缩减，在保证业务Key不重复的情况下，尽量减少Key的长度。

## Redis-Cluster 中使用Multi·Key操作：

当然，当我们使用Redis集群的时候，一些业务场景需要让一些相同业务含义的数据落在同一个节点上，方便我们进行MSET、MGET操作（Redis Cluster跨节点点不支持MSET、MGET操作），这个时候我们可能就需要使用Redis的Hash Tag功能，它允许用key的部分字符串来计算hash；

>当一个key包含 {} 的时候，就不对整个key做hash，而仅对 {} 包括的字符串做hash。

但是这个会带来一个问题，当流量高峰期的时候，大部分的请求都会落到固定的hash槽中，单台Redis Node的压力会非常大，因为Redis是单进程单线程模式，采用队列模式将并发访问变为串行访问，流量超过了单节点的处理能力后，触发Jedis客户端的超时，重连等错误，客户端不断报：


```java
JedisClusterMaxRedirectionsException: Too many Cluster redirections?
```
![](https://gw.alipayobjects.com/zos/skylark/bddb8985-d638-41b8-8a3f-f57eeda4c8c9/2018/png/6bd4929a-22b7-4dd9-a7b6-6fe36cefb957.png)
首先想到最简单的办法就是针对不同的业务场景中的Multi Key操作进行拆分，尽量不要落到同一个Redis Node中，怎么操作让他们落在不同的Redis Node上呢 ？人工通过改变Hash Tag，计算key中{}内的Hash值对应的slot，来达到移动slot的目的，我们称之为[预分桶](http://redis.io/topics/cluster-spec#overview-of-redis-cluster-main-components)。预分桶的实现如下：


```java
HASH_SLOT = CRC16(key) mod 16384
```
其中CRC16算法的C语言实现：[CRC16](http://redis.io/topics/cluster-spec#appendix)，对Crc16的原理可以参考：[Redis源码中的CRC校验码](http://blog.csdn.net/guodongxiaren/article/details/44706613)
Jedis也提供了计算slot的方法：redis.clients.util.JedisClusterCRC16.getSlot()
但是这种方案并没有从根本上解决问题，当一个不可拆分的业务数据量达到单台Redis的瓶颈的时候怎么解决 ？参考唯品会、美团、饿了吗的Redis实践，有一些采用的增加Proxy让服务端分片的方式（[twemproxy](https://github.com/twitter/twemproxy)等），对于multi key操作有服务端进行多次操作来达到目的，但是之前的一次操作会被拆为多个单操作，导致QPS翻倍，好在的是我们可以通过水平扩展节点的方式来达到分担压力的目的。

## 总结：

之所以我们会自主搭建Redis-Cluster，原因是在14年阿里云还未推出KV-Store（云Redis）存储设备，所以只能自建，自建场景下，我们缺乏专业的Redis运维同学，所以导致我们在Redis的运维上除了N多问题，给人的印象是每逢大促必挂，给业务带来了很大的损失。回过头来再计算一下我们购买机器、SSD、运维等的成本，比使用阿里云的Redis还贵，所以如果能使用阿里云的服务还是尽量采用阿里云的服务。


