---
layout: post
title: 我对负载均衡演变过程的理解
categories: Middleware
description:
keywords: lvs 负载均衡
date: 2017-02-10
---
之前看到SLB、VIP、LVS、VIPServer这些很相近的名词总不能很好的进行区分，不知道这些在负载均衡领域具体的使用场景，我期望通过这篇文章，帮大家梳理一下这些名词的区别。我们从一个域名的访问开始，挖掘一下我们的访问链路上到底出现了什么，如果业务更复杂，我们的系统改怎么演变。

### 域名解析：
我们在命令行模式下ping一个域名：popcorn.alibaba-inc.com 
![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/5d19975274487378d0e0b530291bc113.png)

返回的IP是140.205.247.101，在PSP上查询此IP，得到：
![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/a63dcc1b6533ff65bb654691b131f754.png)

可以看到，我们拿到的是LVS的机器，也就是说，域名绑定的机器IP是我们应用专属的LVS的IP地址（DNS域名解析服务）。

对这部分不理解的，可以去尝试使用阿里云服务中的域名服务，申请一个域名，然后进行域名解析服务：
![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/6e39897cb238e853df6c99b472475257.png)

### 负载均衡演化
#### DNS负载均衡
一个域名，我们可以添加多个A记录，绑定后端多台服务器（如果没有SLB的话），这样我们可以利用DNS的负载均衡帮我们实现服务器断的负载均衡。在**无负载均衡的权威 DNS** 中，Local DNS 访问权威 DNS，权威 DNS 会将这 绑定的多个解析记录全部返回给 Local DNS， Local DNS 会将所有的 IP 地址返回给网站访问者，网站访问者的浏览器会随机访问其中一个 IP。
而在**有负载均衡的权威 DNS** 中，网站访问者的请求到来时，权威 DNS 会根据解析记录的权重轮询 全部 A 记录(默认权重 1:1:1)，依次返回 3 个 IP 地址，每次返回一个IP地址；当然用户可以在这里修改A记录的权重值。
![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/fb53a3139a7c9713e7a9a64fa8b5f2cb.png)
我们的网站架构可能是这个样子：
![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/3372b24d510db1616b4e3aa1a7c2cdfb.png)


使用DNS负载均衡的问题是一般DNS解析都会有TTL(TTL指各地DNS缓存您域名记录信息的时间),当后端某个服务器挂掉的时候，由于TTL的缓存得不到及时清除，所以会让部分流量进入到已经宕掉的机器上，造成一定的损失。所以，这个时候，我们最好引入相比于后端应用更加稳定、相比于DNS负载均衡更加灵活的负载均衡器，前置在我们的链路中，比如LVS、F5或者Nginx。

#### 引入官方LVS负载均衡
引入官方LVS负载均衡后，我们的应用架构可能是这样子的：
![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/192ecd3642a05c104ea3b74d753a9995.png)


我们采用[官方的LVS](http://www.linuxvirtualserver.org/)作为软负载，并通过主备的方式达到LVS的容灾策略，LVS与Real Server之间通过heartbeat方式进行健康检查，LVS主备间通过KeepAlived进行状态检测；通过LVS，当我们Real Server 集群有机器上下线时，就不需要与DNS打交道了，只需要与LVS交互，并且LVS本身可具备对RealServer的健康检查，让我们的服务上下线变得更加容易和简单，并且我们可以在LVS中自定义相比于DNS更丰富的负载均衡策略；
这种架构中，同一时刻只有一台LVS对外提供服务，另外一台一直处于stand by 状态，直到提供服务LVS的挂了。在LVS中配置了Real Server的真实IP地址，用于健康检查和负载均衡，在Real Server中配置了Virtual IP地址，用于组件Virtual Server环境（LVS + Real Server统称为Virtual Server）。
我们可以看到Virtual IP 并不像内网或者外网IP那样绑定上去后就固定唯一了，Virtual IP 我们是可以绑定到多台机器上。即使我们LVS采用了主备模式，单点提供服务的LVS还是可能会成为性能瓶颈，无法进行横向扩张，其次，官方LVS缺乏攻击防御功能，在转发模式上，只支持NAT/DR/TUNNEL 三种，在多VLAN 网络环境下部署成本极高。
那么如何解决这些问题呢？阿里的负载均衡设备在官方 LVS 基础上进行了定制化和优化，比如LVS采用集群方式部署，增加攻击防御模块，新增转发模式 FULLNAT，实现 LVS-RealServer 间跨 VLAN 通讯等；
Ali-LVS 的开源地址： https://github.com/alibaba/LVS
虚拟IP和IP漂移：http://xiaobaoqiu.github.io/blog/2015/04/02/xu-ni-iphe-ippiao-yi/
#### 引入Ali-LVS改进版负载均衡
当引入了Ali-LVS后，我们的架构变成了如下结构：
![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/6bd1e2736ac37fae3e88f44cfb71b0ad.png)
我们的LVS变成了集群模式，那么就需要我们感知LVS集群中的机器服务状态并能自动进行健康检查，LVS本身也需要负载均衡；这个时候就需要引入另外一个硬件设备了-**交换机**，LVS和交换机间运行OSPF心跳，1个VIP（Virtual IP）配置在集群的所有LVS上，ECMP负责将数据流分发给LVS集群，当一台LVS宕机后，交换机会自动发现并将其从ECMP等价路由中剔除。外部流量到了交换机后下一步具体走哪条路径是在交换机上配置不同的hash策略控制的，一般是源IP+源端口。

#### 从4层负载到7层负载
当然，这种架构已经能解决我们大部分问题了，但是LVS是在四层协议上实现的负载均衡，我们有一些业务需要SLB实例服务端口使用的是7层HTTP协议，怎么办呢？我们可以引入Tengine，Tengine是当前最流行的7层负载均衡开源软件之一，且已经开源，那么我们的架构可能会演变成以下这个样子：
![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/e25b2a68a163516cdb54481dcd4c136d.png)
客户端访问SLB实例VIP时，相关请求由SLB实例对应的LVS集群处理，如果相应的SLB实例服务端口使用的是4层协议（TCP或UDP），那么LVS集群内每个节点都会根据SLB实例负载均衡策略，将其承载的服务请求按策略直接分发到后端Real Server服务器，并同时维护会话保持等特性；
如果相应的SLB实例服务端口使用的是7层HTTP协议，那么LVS集群内每个节点会先将其承载的服务请求均分到Tengine集群；而后，Tengine集群内的每个节点再根据SLB实例负载均衡策略，将服务请求按策略最终分发到后端Real Server服务器，并同时维护会话保持等特性。


针对LVS的工作原理可以参考：[LVS基础](http://www.atatech.org/articles/48739) 、[LVS及相关网络概念](http://www.atatech.org/articles/48571) 或 [负载均衡（SLB）技术原理浅析](http://www.atatech.org/articles/50329)

#### 从公网访问负载到内部访问负载
我们通过LVS集群和Tengine集群解决了公网到阿里内部应用的负载均衡，随着阿里内部很多应用的集群逐渐扩大，这些应用在被访问的时候也是需要被进行负载均衡（我们默认已包含状态检查模块）的，那么我们把应用访问方式大致分为两类，一类是基于HSF的RPC远程服务调用，另外一种是基于HTTP的服务调用；基于HSF的服务调用的负载均衡是由ConfigServer完成，这部分可以参考ConfigServer的实现；基于HTTP的负载均衡是VIPServer，通过VIPServer，我们内部应用通过域名范围内部应用时，就没有必要重新走一遍公网的DNS解析，链路顶端的各种负载均衡了，通过VIPServer，我们可以与内部应用进行直连，相当于对我们的访问链路进行了优化和加速，当然，VIPServer在对HTTP的内部调用上除了可以做负载均衡、健康检查外，还做了异地容灾的调用实现、流量的控制、环境管理、灰度的发布等，有关VIPServer的功能可以参考VIPServer的功能介绍。那么，在引入这两个设备后，我们的应用架构可能会演变成这个样子：
![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/2e9d70180eda8b4bb88b13c27ee33cd4.png)


参考：

https://xue.alibaba-inc.com/trs/doc/learningDocDetail.htm?spm=a1z2e.8101737.list.8.aRav7f&docUid=770f4104-0d89-4a5f-8237-f112c61fb006
http://www.atatech.org/articles/50329





