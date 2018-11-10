---
layout: post
title: DevOps的三种方式
categories: Program
description:
keywords: devops
date: 2017-12-24
---

## 前言
这篇博客中提到的“三种方式“源自 [《DevOps Handbook》](http://www.itrevolution.com/book/the-devops-handbook/) 及[《凤凰项目》](https://itrevolution.com/books/novel/)（The Phoenix Project: A Novel About IT, DevOps, and Helping Your Business Win.），这三种方式描述了构成 DevOps 的理论框架、流程、实践及价值观和哲学。

感谢《Lean IT》的作者 Mike Orzen 为此文提供宝贵建议。

## 三种方式
下文将介绍三种模式及在该种模式指导下的 DevOps 实践。

### 第一种方式： 系统思考
![](https://gw.alipayobjects.com/zos/skylark/90fda8c1-ed1d-414d-bf7a-379e3fc7bd54/2018/png/46243fa7-bffa-4015-acd8-2597e974feee.png)

第一种方式强调全局优化，而非局部改进。— 大到部门职能划分（例如研发部和运维部门），小到个人（开发和系统工程师）。

这种方式将关注点放在整个业务价值流上。换句话说，整个团队应该关注在从需求被定义到开发，再到运维这个过程，直到价值被以服务的形式交付给最终用户。

将这种方式带到实践中的产出便是永远不要将已知的缺陷传递到下游工作，永远不要为了局部优化影响了整体价值流交付，总是为了增加价值流动努力，永远追求对架构的深刻理解。

涉及到这种方式的实践有：

* 所有环境和代码使用同一个仓库，将软件包纳入版本管理
* 团队共同决定发布流程
* 保持 DEV、TEST、PRODUCTION 环境的一致性
* 自动化回归测试
* 小步提交，每日部署；而不是一次部署大量变更
* 更快、更频繁发布

### 第二种方式：经过放大的反馈回路
![](https://gw.alipayobjects.com/zos/skylark/f6cc08b5-b7e5-4162-92a3-0d0df213d902/2018/png/c8e97809-c293-4170-a84c-143789ea7faf.png)

第二种方式是创建从开发过程下游至上游的反馈环。几乎所有的流程改进都是为了从时间上缩短和从覆盖面上放大反馈循环，从而可以不断地进行必要的改正。

第二种方式的产出是关注到价值流中所有涉及到的用户，包括价值流内部和外部的，缩短和放大反馈回路，并且可以随时定位到需要改进的地方。

涉及到这种方式的实践有：

* 代码审查及配置变更检查
* 有纪律的自动化测试，使许多同时的小型敏捷团队能够有效地工作
* 尽早设置监控预警
* 修复 bug 为团队最高优先级
* 团队成员之间高度互相信任
* 团队之间保持沟通和良好合作
### 第三种方式：持续做试验和学习的文化
![](https://gw.alipayobjects.com/zos/skylark/dfea288e-e3e5-4f41-b1c0-998f7def0202/2018/png/096c5099-06da-403c-8329-3c88f8851fd9.png)

第三种方式提倡持续做试验，承担风险、从失败中学习；通过反复实践来达到精通。

我们需要实验和冒着失败的风险，及时不断地尝试将我们置于一个危险的境地，我们要通过反复试错来掌握使我们远离危险的技能。

第三种方式的输出为为改善日常工作分配时间、奖励团队冒险精神，将错误人工引入系统以提高系统健壮性。

最具有代表性的就是 Netfilx 的 Chaos monkey ，Netflix 在他们的生产环境搭建一个服务用于定时随机关闭服务器，用以模拟服务器正常损坏或服务异常，他们的系统长期在这种环境下运行，“服务器故障”成为系统每日都要面临的问题，因此当服务器真的以外故障时不会对系统整体造成任何的影响。

## 译者后记
全局优化、快速反馈、鼓励失败。我们发现其实敏捷、精益、持续交付、DevOps中间有很多相似的东西。

> 参考资料: https://es.slideshare.net/SonatypeCorp/devops-connect-josh-corman-and-gene-kim-discuss-devopssec

> http://itrevolution.com/the-three-ways-principles-underpinning-devops/

转发自：@duyidong

