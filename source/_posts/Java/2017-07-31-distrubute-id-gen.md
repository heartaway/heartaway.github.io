---
layout: post
title: Distributed System Id Gen
categories: Java
description:
keywords: id-gen
date: 2017-07-31
---

# 分布式系统高效唯一ID生成方案
>背景：在分布式系统中，对于订单ID如何保证全局唯一地高效的生成，并对性能影响不大的情况下，利于建立索引和业务使用？


### 方案一：使用数据库主键ID
优势：实现简单；
弊端：使用数据库增加写库的压力，生成订单的上限受限于数据库的写性能上限；扩展性差；

### 方案二：使用数据库的列+1
主键采用业务编码，普通列current_value标识id当前值，通过数据库的行锁来保障唯一的；
优势：一张表可以存储不同业务类型的生成规则，没必要每一个id规则都新建一张附表；
劣势：涉及行锁，性能不高；

需要注意使用此方式生成数字序列事务隔离级别需要是RR。

### 方案三：单点批量生成订单号的服务
优势：批量降低了数据库的读写压力
弊端：服务单点风险；

### 方案四：使用UUID
优势：本地生成ID，不进行远程调用，延迟低；
弊端：无法保证趋势增长；uuid过长且建立索引效率低下；64位太长；

### 方案五：使用毫秒数
优势：本地生成ID，不进行远程调用，延迟低；ID为整数利于建立索引；
弊端：并发超过1000，可能会重复；
优化方案：使用LRU Cache存储最近生成的100Key，去除后先判断是否重复；进一步降低重复的可能性；但是无法根本上解决高并发下的重复；

### 方案六：订单号分段表示具体含义
方案描述：使用39bit表示毫秒数、4bit表示业务线、7bit表示机器等
优势：实现简单，且重复可能性很低；
弊端：订单长度过长，不利于业务使用；

### 方案七：使用Redis服务+毫秒数双重方案
优势：使用redis的自增序列特性，每次获取一定数量的数据（1000个），降低了每次调用Redis的开销；当Redis挂了，使用基于毫秒数的本地生成ID优化方案；
弊端：redis挂了后，订单号的重复性不可避免；

使用redis需要配置主备，避免在极端情况下redis节点down机， 导致丢失序列或序列重复。

### 总结：
我们在设计分布式全局唯一单号的时候，不仅需要考虑生成单号的是否重复性，还要考虑生成时的性能开销、是否有容灾方案、是否便于建立索引、是否利于业务使用等。基于这些点的考虑，我们采用了方案六作为线上的唯一生成方案，并把此方案作为一个业务隔离的二方库，可以提供给不同业务方式用；



参考：[高可用高性能可扩展的单号生成方案](https://mp.weixin.qq.com/s?__biz=MzIwODA4NjMwNA==&mid=2652898525&idx=1&sn=7e7c1a64fa288eceb17fd4057afd2a62&chksm=8cdcd092bbab598469534293f1eff0d65edcfeaf65050c79f8d054e008c0d1843af1502d34a7&mpshare=1&scene=1&srcid=0731m6EiK3LyvkmVltj03hG9&key=807bd2816f4e78936c683f90def8c012cc01115330c60aaa1377edae1f76fec7b00e4698f4d6ddc22680f9a80dade15c2cf6caa00853e61b925ff2ab93d1110eff532aedcb1347a8db1074bea22542a1&ascene=0&uin=MTE3NTkxNDk4MA%3D%3D&devicetype=iMac+MacBookPro11%2C4+OSX+OSX+10.12.5+build(16F73)&version=12020810&nettype=WIFI&fontScale=100&pass_ticket=Q2dSEf7cwUT%2FHuuZ1wpxUh3%2Fd9hovwWFg9YsoOzUMokJ2WygwbBZxUPkJXDCEA09)




