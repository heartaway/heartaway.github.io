---
layout: post
title: 系统权限控制体系
categories: Program
description:
keywords: permission system 权限 权限控制
date: 2018-01-04
---
在 Web 应用开发中，安全一直是非常重要的一个方面。安全虽然属于应用的非功能性需求，但是应该在应用开发的初期就考虑进来。比如我们开放的功能页面需要登录授权之后才能访问，一些功能需要具备特定权限的人才能操作；再比如我们开放了数据API接口，如果不做访问控制，那么任何人都可以调用，当被不法分子操作时将给我们带来巨大的麻烦。那么在Java 整个体系中访问控制是否有一套理论技术支撑呢，我们是否可以做一个通用性的访问控制系统来完成分布式系统架构下的复杂的权限控制？接下来会一一介绍。

### 访问控制的本质：
系统权限控制 本质上是访问控制（Access Control），那访问控制的本质又是什么呢？其实就是合法的访问受保护的资源，通俗的解释就是“【谁】是否有可以对某个【资源】进行某种【操作】”；可以看出访问控制的三个基本要素：主体（请求实体）、客体（资源实体）、控制策略（属性集合）；

### 访问控制需要完成的两个任务：
识别和确认访问系统的用户；
决定该用户可以对某一系统资源进行何种类型的访问；

### 访问控制理论模型：
* DAC&MAC模型
    * DAC：自主访问控制；
    * MAC：强制访问控制，一般用于多级安全军事系统；
* IBAC模型：
    * 基于身份的访问控制模型
    * 举例：登录验证
    * 比如Java中使用cookie、session存储回话标识；
* RBAC模型：
    * 基于角色的访问控制（Role-Based Access Control）
    * 用户、角色、权限
    * RBAC是ABAC的一种单属性特例；
    * 1992年David F.Ferraiolo & D.Richard Kuhn在第十五届国家计算机安全会议上提出；
    * 论文：https://csrc.nist.gov/projects/role-based-access-control
    * 举例：丰趣-小二后台的认证授权模型设计；
    * Spring Security、Apache Shiro、Ali ACL
* ABAC模型：
    * 基于属性的访问控制模型 (Attribute Based Access Control)
    * 举例：阿里云、AWS；
    * 论文：
        * https://csrc.nist.gov/projects/attribute-based-access-control
        * https://link.springer.com/chapter/10.1007/978-3-319-25645-0_14
        * http://profsandhu.com/dissert/xin_slides.pdf

示意图：
![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/068d3b155b6538ee6f4651d7febe19cd.png)
基于RBAC权限模型的常见数据库模型设计：
![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/af745d65be9e6922518ff6402e805464.png)
设计的核心主题: 用户、权限、角色、用户角色、角色权限、用户组、用户组角色、操作审计
阿里巴巴登录鉴权与审计的三驾马车：BUC、ACL、OpLog；

### Java常用访问控制框架：
* JAAS框架：
    * Authentication（鉴别），    Authorization（授权），Accounting（计费）；
    * Java认证和授权服务（Java Authentication and Authorization Service，简称JAAS）
    * 支持的框架：LDAP
    * 特点：面世时间早，使用受限，不建议使用；
* Spring Security框架
* Apache Shiro框架

### 权限系统的演变历史：
#### 1： 标准的JAAS 时代；
J2ee时代，Java提出了标准的鉴权服务，即jaas；通过简单的容器配置和文件配置，通过一个LDAP（可以用数据库，只是效率不高），就可以提供一个极为高效便捷的权限管控服务。这个模式不仅支持页面管控，还支持ejb服务接口管控。其鉴权因为ldap的数倍于数据库的查询效率而无需任何缓存，速度很快。但是伴随着分布式服务化进程，应用的数量无限度增长，这种散落在各个容器的配置给容灾和修改，都带来了极大的挑战，ldap的可读化差，修改和编辑极为不便，当需求一旦个性化超过了树能够表达的模型便很难在适应。并且当ldap的数据爆炸式增长，且呈现28规律时（数据冷热不均），或者如果需要频繁的写ldap，查询效率会陡然下降。虽然这种方式目前并不流行的，但是由于历史原因，还存留着使用这种方式的管控方式，所以我们在Spring Security或者阿里的ACL中都还能看到对JAAS的支持。
#### 2： 单点登录(SSO)+接口鉴权时代；
把分散在各个系统的登录认证服务统一到一个系统中来，统一管控登录授权业务，用户只要在一个系统中登录了，在其它系统中就没必要再次登录了，这就是SSO。简单的实现是登录授权系统部署在一台机器上，不涉登录系统的多机部署，此架构具有单点风险；任何具备高可用思维的架构师都不会允许此风险存在，原因有二：1. 统一登录中心后，SSO成为极为核心的应用，如果SSO系统挂了，那么需要登录的任何服务器都无法正常提供服务；2. 单台机器不具备抵抗登录风暴的能力； 所以SSO系统必须成为集群部署模式。其次，在访问控制模型上，也必须放弃JAAS方式，转而使用RBAC模型；
#### 3： 统一登录（分布式Session） + 接口鉴权时代；
SSO系统集群部署后，面临的首要问题就是Session的共享问题，比如用户在sso-1 机器上登录了，下次访问sso-2机器时，也必须是登录态的。分布式Session使用较多的方案为：Session集中管理；比如阿里巴巴基于Tair 缓存体系的共享session体系tbsession。如果采用了session + cookie的方案，并且服务端集群是多域名共享登录的话，那么还需要提供cookie跨域同步的能力（解决cookie不能跨域的问题）。

