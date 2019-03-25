---
layout: post
title:  Mybatis 核心理念
categories: Java
description:
keywords: mybatis,核心理念
date: 2018-04-09
---

>Mybatis的本质：混合型Java对象持久化工具；

### 前置说明：
需要提前具备SQL、JDBC、RDBMS、XML、OOP等知识；


### 什么是Mybatis：
核心点：

* 支持了SQL、存储过程、对象/关系映射的持久化框架；
* 消除了几乎所有的JDBC代码和需要手工处理的参数和结果；
* 支持XML和注解两种方式来配置sqlmap、mapper、pojo三者之间的关系。

Mybatis理念：坚信SQL、RDBMS将继续使用**30年**；：<!-- more -->

### Mybatis 核心概念
数据模型：

* Configuration
* SqlSessionFactory
* SqlSession
* MappedStatement

运行期四大组件：

* Executor 
* StatementHandler 
* ParameterHandler 
* ResultSetHandler

算法：

* Java静态&动态代理
* 待补充



### Mybatis 发展历程
ibatis → mybatis

ibatis 于2002年由 Clinton Begin创建；
ibatis 于2010年暂停维护，专有apache维护，改名为mybatis，版本定位mybatis 3；

### Mybatis 工作流程

![](/images/posts/20180409/mybatis-work-flow.jpg)
​    （选自CSDN：亦山）


### Mybatis 中的设计思想
1. 外部化SQL&参数配置化，将设置与运行策略分离；
2. 采用配置化原因：降低编码复杂度，提高可读性和可维护性；
3. 封装SQL；基于接口编程，屏蔽SQL对外部的具体实现；
4. 随处可见的工厂模式、builder模式、Proxy模式、装饰器模式；
5. 将大系统设计为多个子系统，每个子系统的功能相对集中，尽可能将那些需要由不同的开发角色处理的任务分离开来。
6. Cache的实现模式：链式静态代理，原始对象PerpetualCache；
    * 代理模式三要素：共同接口，真实对象、代理对象（装饰器模式）；
    * 我们可以采用对象工程屏蔽代理类的生成；
    * 静态代理的本质：不侵入代码的情况下，扩展原对象功能；
7. Plugin实现采用Java动态代理方式，责任链模式实现；
8. 基于注解或者路径的自动扫描；


### Mybatis主要功能
* 使用XML或Java API配置Mybatis
* 使用XML或注解配置SQL映射器（Mapper）
* 基于OGNL表达式的动态SQL构建
* 事务支持


##### Executor层次
![](/images/posts/20180409/mybatis-executor-topo.jpg)

CachingExecutor 是正式Executor实现类的一个代理wrapper。

##### 动态SQL
4个核心关键词:

1. if 
2. choise(when,otherwise)
3. trim（where,set）
4. foreach

##### 缓存体系
1. 本地缓存；
    作用域：SqlSession内数据本地缓存；
    数据结构：PerpetualCache （HashMap）
    作用：利用本地缓存机制（Local Cache）防止循环引用（circular references）和加速重复嵌套查询。
    配置： localCacheScope=STATEMENT|SESSION设置作用域，建议使用STATEMENT，防止分布式环境下导致的脏读；

2. 二级缓存
    作用域：namespace；SqlSession间数据缓存；
    默认底层缓存结构：LruCache包装的PerpetualCache 
    选择：在分布式环境下，建议使用外部分布式缓存系统，而非Mybatis的本地缓存；


##### 作用域：scope
* SqlSessionFactory 建议应用级别；   
* SqlSession 请求或方法级别（非线程安全）；
* Mapper Instance 方法级别；

#### 与Spring集成
下回详解；

官方文档:http://www.mybatis.org/spring/index.html

版本说明：http://www.mybatis.org/spring/index.html


#### 工程示例：
大家可以基于测试工程自行联系相关设计理念以及工作原理：

[https://github.com/heartaway/mybatis-study-demo](https://github.com/heartaway/mybatis-study-demo)


#### 参考文档:
http://www.mybatis.org/mybatis-3/zh/configuration.html

http://www.mybatis.org/spring/index.html

https://blog.csdn.net/column/details/mybatis-principle.html

《iBatis实战》





