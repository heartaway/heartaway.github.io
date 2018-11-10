---
layout: post
title: AWS应用自动发现服务
categories: HighAvailability
description:
keywords: aws application-discovery 自动发现
date: 2018-03-15
---

>定义：AWS Application Discovery Service 收集并呈现数据，以使企业客户能了解其本地环境中服务器的配置、使用和行为。



## 工作模式：
### 无代理自动发现
原理：基于 VMware vCenter Server 的环境的数据上报；

弊端：

* 采用VMware 方式在国内并不常用；
* 采集到的数据有限，只是系统层面的静态数据与CPU、MEM的指标数据；


### 基于代理(Agent)的发现模式：
通过Agent可以采集更加丰富的数据集合，比如系统进程，系统间的网络连接；这里重点了解Agent模式。

## 采集的数据：
* 静态配置数据： 
    * 系统标识信息：主机名、IP地址、MAC地址、操作系统名称、操作系统版本号，cpuType；
    * 运行中的进程数据；
    * 系统间的网络调用（TCP/IP v4 和 v6）；
        * sourceIP、sourceProcess、destinationIp、destinationPort、dstProcess，ipVersion；
* 性能指标数据：
    * 操作系统层面：
        * 参考之前的OS监控采集项；
    * 进程层面：
        * %CPU
        * %MEM
        * %DISK  
    * 网络层面：
        * 待补充  


### 数据安全性：
采用SSL协议进行数据传输；
数据存储采用AWS KMS静态加密；

## Agent模式下的自动发现
Agent启动时，自动注册到Arsenal中，并频繁对该服务执行 ping 操作以获取配置信息。
![](https://gw.alipayobjects.com/zos/skylark/2a647cf5-d9b2-42f6-bcab-39a16b5ead2b/2018/png/179f1e58-a729-4ab0-8dea-b604269d0ad9.png)

### 数据收集
数据的收集需要自动显示开启；

#### 数据采集状态：

* STARTED – 收集工具已开始收集和发送数据到 Discovery Service。
* START_SCHEDULED – 已计划数据收集开始时间。下次收集工具联系 AWS 时，它将开始将数据发送到 Discovery Service，并且收集状态将更改为 STARTED。
* STOPPED – 收集工具已停止发送数据到 Discovery Service。
* STOP_SCHEDULED – 已计划数据收集停止时间。下次收集工具联系 AWS 时，它将停止向 DiscoveryService 发送数据，并且收集状态将更改为 STOPPED。

### 组件分类
DiscoveryService识别组件后，通过TAG标签的方式，对资源进行标记；用户也可以同通过控制台进行自定义TAG标记；

**进程数据分类方法：** 通过采集到的进程数据，可以推测出aws判断进程属于哪一类基础设施应该是通过进程名称与commondLine来判断的。

### Agent模式自动发现不足：
* 不能保存资源组件快照或跟踪资源变更；
* 收集的性能数据不是通用的运行状况监控解决方案；

## 自动发现API
1. Agent相关API
    *  Agent的启动、停止数据采集操作；
    *  Agent的描述信息；
    *  Agent的运行状态；
2. 配置相关API
    *  server、application、process、connection 的指标查询、过滤与配置；
3. 标签相关API
    * TAG的增删改查； 

## 其他：
产品链接：[https://aws.amazon.com/cn/documentation/application-discovery/](https://aws.amazon.com/cn/documentation/application-discovery/)

API地址：[https://docs.aws.amazon.com/zh_cn/application-discovery/latest/APIReference/discovery-api.pdf#discovery-api-queries](https://docs.aws.amazon.com/zh_cn/application-discovery/latest/APIReference/discovery-api.pdf#discovery-api-queries)

