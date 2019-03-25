---
title: 一次线上系统响应慢的问题定位
date: 2019-03-25 17:17:37 +0800
tags: [问题定位, 稳定性]
categories: 稳定性
---

<a name="fa77ab6c"></a>
### 发现问题
3月8日收到线上部分区域监控系统报警，提示数据处理服务大批异常报警；登录机器查看系统负载情况：<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/104361/1553507051995-7cce4dc5-7479-4a82-bf77-a8b48fb4928d.png#align=left&display=inline&height=241&name=image.png&originHeight=245&originWidth=722&size=135506&status=done&width=711)<br />系统CPU占用并不高。

同事提示，发现了HSF日志中输出了HSF连接池满的情况：<br /><!--more--><br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/104361/1553520952926-7421d58c-8fe0-45ff-a091-5bad1bc04a8f.png#align=left&display=inline&height=288&name=image.png&originHeight=576&originWidth=1428&size=450665&status=done&width=714)

第一反应是后端处理太慢导致，查看了数据处理耗时统计：<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/104361/1553520978790-be1d0a79-07eb-408a-b261-b9be01186024.png#align=left&display=inline&height=375&name=image.png&originHeight=750&originWidth=1384&size=331071&status=done&width=692)<br />发现峰值耗时已经超过了160秒；

<a name="0aff8430"></a>
#### 处理方案：
暂且为定位根因，先做了服务器重启；重启后恢复，以为是偶发情况；


<a name="258c8519"></a>
### 介入排查
3月12 ~ 15日之间 线上Dataprocess的报警频次越来越高，且数量越来越大。在3月15日进行系统问题定位于排查；<br />分析代码后，做了一下优化：
1. 接受日志处理的线程池的队列之前采用的是ArrayBlockingQueue，改用双锁队列LinkedBlockingQueue；
1. 线程池拒绝策略采用自定义的DiscardPolicy，不再使用DiscardOldestPolicy（会进行队列poll操作，需要竞争锁）；
1. 增加了线程池监控，监控线程池队列大小、处理任务数、丢弃任务数；


<a name="722d8e0f"></a>
### 具体定位
<a name="52e514c1"></a>
#### 指标观察：
系统发布后，问题并未得到解决，报警依旧，采用tsar 观察系统各项指标：<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/104361/1553507935197-8e95d494-47bf-4d22-97d6-548945fd18ea.png#align=left&display=inline&height=300&name=image.png&originHeight=384&originWidth=956&size=119399&status=done&width=746)<br />（系统load，对于8core cpu，load并不高）<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/104361/1553507967263-1ff8e52b-aafc-4b58-921d-947f3a7d3125.png#align=left&display=inline&height=244&name=image.png&originHeight=370&originWidth=1130&size=176313&status=done&width=745)<br />（系统剩余内存不足）<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/104361/1553508026297-6ee1935f-98f8-43f8-83b2-42da9faeeed2.png#align=left&display=inline&height=151&name=image.png&originHeight=368&originWidth=1822&size=168486&status=done&width=746)<br />（系统的io操作并不频繁）

开始怀疑jvm参数配置有错误，查了jvm的堆内存分配，从参数和合理上看并没有太大问题。<br />堆分配如下：<br />启动时堆的初始化大小： 10G（-Xms）<br />堆最大值为： 10G（-Xmx）<br />年轻代空间大小：4G（-Xmn）<br />初始持久代：256m （-XX:MetaspaceSize）<br />最大持久代：512m （-XX:MaxMetaspaceSize）

<a name="b5a52648"></a>
#### Jstack分析：
开始对线上系统进行jstack，分析线程栈具体卡在什么地方；<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/104361/1553510300293-c8173e97-09fd-41a5-ab38-c8c91976834f.png#align=left&display=inline&height=443&name=image.png&originHeight=885&originWidth=2000&size=514798&status=done&width=1000)

发现  69%的线程处于blocked状态；<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/104361/1553510331156-ccad2748-8d71-4b8a-accc-62d6acdfaaf8.png#align=left&display=inline&height=375&name=image.png&originHeight=749&originWidth=2000&size=462845&status=done&width=1000)<br />从线程分组上看，HSFBizProcessor基本全部处于Blocked状态，log-handler-thread线程所有都处于blocked状态；

![image.png](https://cdn.nlark.com/yuque/0/2019/png/104361/1553510391861-aa83d7a9-85af-4980-94e0-0876c73d26ed.png#align=left&display=inline&height=142&name=image.png&originHeight=284&originWidth=2000&size=335659&status=done&width=1000)

查看处于blocked的HSF线程栈信息：

```
HSFBizProcessor-DEFAULT-12-thread-1673 - priority:10 - threadId:0x0000000006d32000 - nativeId:0x4f72 - nativeId (decimal):20338 - state:BLOCKED
stackTrace:
java.lang.Thread.State: BLOCKED (on object monitor)
at java.util.Arrays.copyOf(Arrays.java:3181)
at java.text.DateFormatSymbols.getShortMonths(DateFormatSymbols.java:433)
at java.util.Calendar.getFieldStrings(Calendar.java:2241)
at java.util.Calendar.getDisplayName(Calendar.java:2111)
at java.text.SimpleDateFormat.subFormat(SimpleDateFormat.java:1125)
at java.text.SimpleDateFormat.format(SimpleDateFormat.java:966)
at java.text.SimpleDateFormat.format(SimpleDateFormat.java:936)
at java.text.DateFormat.format(DateFormat.java:345)
at ch.qos.logback.core.util.CachingDateFormatter.format(CachingDateFormatter.java:48)
- locked <0x00000006223577e8> (a ch.qos.logback.core.util.CachingDateFormatter)
at ch.qos.logback.classic.pattern.DateConverter.convert(DateConverter.java:61)
at ch.qos.logback.classic.pattern.DateConverter.convert(DateConverter.java:23)
at ch.qos.logback.core.pattern.FormattingConverter.write(FormattingConverter.java:36)
at ch.qos.logback.core.pattern.PatternLayoutBase.writeLoopOnConverters(PatternLayoutBase.java:115)
at ch.qos.logback.classic.PatternLayout.doLayout(PatternLayout.java:141)
at ch.qos.logback.classic.PatternLayout.doLayout(PatternLayout.java:39)
at ch.qos.logback.core.encoder.LayoutWrappingEncoder.encode(LayoutWrappingEncoder.java:115)
at ch.qos.logback.core.OutputStreamAppender.subAppend(OutputStreamAppender.java:230)
at ch.qos.logback.core.rolling.RollingFileAppender.subAppend(RollingFileAppender.java:235)
at ch.qos.logback.core.OutputStreamAppender.append(OutputStreamAppender.java:102)
at ch.qos.logback.core.UnsynchronizedAppenderBase.doAppend(UnsynchronizedAppenderBase.java:84)
at ch.qos.logback.core.spi.AppenderAttachableImpl.appendLoopOnAppendersForStructLog(AppenderAttachableImpl.java:54)
at ch.qos.logback.classic.Logger.appendLoopOnAppenders(Logger.java:273)
at ch.qos.logback.classic.Logger.callAppenders(Logger.java:260)
at ch.qos.logback.classic.Logger.buildLoggingEventAndAppend(Logger.java:424)
at ch.qos.logback.classic.Logger.filterAndLog_0_Or3Plus(Logger.java:386)
at ch.qos.logback.classic.Logger.info(Logger.java:582)
....
at sun.reflect.GeneratedMethodAccessor134.invoke(Unknown Source)
at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
at java.lang.reflect.Method.invoke(Method.java:498)
at com.taobao.hsf.remoting.provider.ReflectInvocationHandler.handleRequest0(ReflectInvocationHandler.java:84)
at com.taobao.hsf.remoting.provider.ReflectInvocationHandler.invoke(ReflectInvocationHandler.java:164)
at com.taobao.hsf2dubbo.DubboServerFilterAsyncInvocationHandlerInterceptor.invoke(DubboServerFilterAsyncInvocationHandlerInterceptor.java:54)
at com.taobao.hsf.invocation.filter.RPCFilterBuilder$TailNode.invoke(RPCFilterBuilder.java:280)
at com.taobao.hsf.debug.DebugServerFilter.invoke(DebugServerFilter.java:25)
at com.taobao.hsf.invocation.filter.RPCFilterNode.invoke(RPCFilterNode.java:71)
at com.taobao.hsf.plugins.spas.SpasServerFilter.invoke(SpasServerFilter.java:143)
at com.taobao.hsf.invocation.filter.RPCFilterNode.invoke(RPCFilterNode.java:71)
at com.taobao.hsf.edas.tps.component.WhiteListServerFilter.invoke(WhiteListServerFilter.java:67)
at com.taobao.hsf.invocation.filter.RPCFilterNode.invoke(RPCFilterNode.java:71)
at com.taobao.hsf.tps.component.TPSServerFilter.invoke(TPSServerFilter.java:68)
at com.taobao.hsf.invocation.filter.RPCFilterNode.invoke(RPCFilterNode.java:71)
at com.taobao.hsf.edas.tps.component.TPSServerFilter.invoke(TPSServerFilter.java:67)
at com.taobao.hsf.invocation.filter.RPCFilterNode.invoke(RPCFilterNode.java:71)
at com.taobao.hsf.invocation.stats.InvocationStatsServerFilter.invoke(InvocationStatsServerFilter.java:47)
at com.taobao.hsf.invocation.filter.RPCFilterNode.invoke(RPCFilterNode.java:71)
at com.taobao.hsf.monitor.log.filter.MonitorLogServerFilter.invoke(MonitorLogServerFilter.java:45)
at com.taobao.hsf.invocation.filter.RPCFilterNode.invoke(RPCFilterNode.java:71)
at com.taobao.hsf.profiler.ProfilerServerFilter.invoke(ProfilerServerFilter.java:32)
at com.taobao.hsf.invocation.filter.RPCFilterNode.invoke(RPCFilterNode.java:71)
at com.taobao.hsf.plugins.eagleeye.EagleEyeServerFilter.invoke(EagleEyeServerFilter.java:66)
at com.taobao.hsf.invocation.filter.RPCFilterNode.invoke(RPCFilterNode.java:71)
at com.taobao.hsf.rpc.server.MethodAbsenceFilter.invoke(MethodAbsenceFilter.java:48)
at com.taobao.hsf.invocation.filter.RPCFilterNode.invoke(RPCFilterNode.java:71)
at com.taobao.hsf.rpc.server.EchoFilter.invoke(EchoFilter.java:45)
at com.taobao.hsf.invocation.filter.RPCFilterNode.invoke(RPCFilterNode.java:71)
at com.taobao.hsf.rpc.generic.GenericInvocationServerFilter.invoke(GenericInvocationServerFilter.java:106)
at com.taobao.hsf.invocation.filter.RPCFilterNode.invoke(RPCFilterNode.java:71)
at com.taobao.hsf2dubbo.DubboGenericServerFilter.invoke(DubboGenericServerFilter.java:36)
at com.taobao.hsf.invocation.filter.RPCFilterNode.invoke(RPCFilterNode.java:71)
at com.taobao.hsf.rpc.server.ServiceAbsenceFilter.invoke(ServiceAbsenceFilter.java:47)
at com.taobao.hsf.invocation.filter.RPCFilterNode.invoke(RPCFilterNode.java:71)
at com.taobao.hsf.common.filter.CommonServerFilter.invoke(CommonServerFilter.java:24)
at com.taobao.hsf.invocation.filter.RPCFilterNode.invoke(RPCFilterNode.java:71)
at com.taobao.hsf2dubbo.context.DubboRPCContextServerFilter.invoke(DubboRPCContextServerFilter.java:48)
at com.taobao.hsf.invocation.filter.RPCFilterNode.invoke(RPCFilterNode.java:71)
at com.taobao.hsf.context.RPCContextServerFilter.invoke(RPCContextServerFilter.java:39)
at com.taobao.hsf.invocation.filter.RPCFilterNode.invoke(RPCFilterNode.java:71)
at com.taobao.hsf.invocation.filter.RPCFilterBuilder$HeadNode.invoke(RPCFilterBuilder.java:246)
at com.taobao.hsf.invocation.filter.FilterInvocationHandler.invoke(FilterInvocationHandler.java:28)
at com.taobao.hsf.remoting.provider.ServerContextInvocationHandler.invoke(ServerContextInvocationHandler.java:35)
at com.taobao.hsf.remoting.provider.ProviderProcessor.handleRequest(ProviderProcessor.java:52)
at com.taobao.hsf.io.remoting.hsf.message.HSFServerHandler$1.run(HSFServerHandler.java:177)
at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:627)
at java.lang.Thread.run(Thread.java:882)
```


发现block在transport-api 包中的hanler方法中，查看代码：<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/104361/1553510475994-942259f9-f93a-4089-8510-1007c06a1cd4.png#align=left&display=inline&height=346&name=image.png&originHeight=692&originWidth=1788&size=680633&status=done&width=894)

发现是日志输出中CachingDateFormatter的SimpleDateFormat 获得的锁未释放。

<a name="71b33d03"></a>
#### 优化方案：
* 取消transport-api中的日志输出，认为此没有必要，且输出的日志量巨大；
* 打印日志优化：日志格式优化-减少不重要信息，降低系统调用，日志采用AsyncAppender包装RollingFileAppender，使用异步日志输出；

![image.png](https://cdn.nlark.com/yuque/0/2019/png/104361/1553521107922-304a7a22-3f14-4920-bb2a-0f2f802cc3fe.png#align=left&display=inline&height=374&name=image.png&originHeight=748&originWidth=1478&size=881357&status=done&width=739)

代办事项：排查打日志导致的锁未释放的真正原因。

发布系统后，继续观察，重启后的一段时间，系统表现正常，但是一段时间后，又有报警出来：<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/104361/1553520842927-d75f27b2-8d65-4122-838a-637fe8a8f06d.png#align=left&display=inline&height=325&name=image.png&originHeight=650&originWidth=1292&size=232677&status=done&width=646)

初步怀疑跟jstack有关，但是jstack时间是 18点40分左右，报警是在19点整，对不上。

从网关侧看到此区域的qps情况，波动非常大；<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/104361/1553520885065-2f331c65-4c83-485a-800e-624cf259b34f.png#align=left&display=inline&height=238&name=image.png&originHeight=476&originWidth=1500&size=331575&status=done&width=750)

<a name="9e497c48"></a>
#### 初步诊断： 
由于没有限流，北京的量太大，历史堆积太对，导致长时间堆积的数据上来后直接把系统打爆了，agent侧没有做流控，服务侧接受方也没有做流控导致的。

<a name="915db64d"></a>
#### 解决方案：
* 采用AHAS 流控降级服务增加限流，对于历史数据直接丢弃，对于新请求增加流控；


第二天，从管控后台上看到北京SLA剧烈波动:

![image.png](https://cdn.nlark.com/yuque/0/2019/png/104361/1552975918909-f06b3f7c-0b3e-4b2b-ba67-a95fa643e574.png#align=left&display=inline&height=352&name=image.png&originHeight=704&originWidth=2246&size=340212&status=done&width=1123)

对此区域一台机器进行offline，然后jstack线程栈继续分析：

![image.png](https://cdn.nlark.com/yuque/0/2019/png/104361/1552976809337-e3bdae4a-7539-433a-9dd6-4261c6527c8b.png#align=left&display=inline&height=220&name=image.png&originHeight=440&originWidth=2178&size=332568&status=done&width=1089)<br />提取hsf的jstack，发现，系统中 业务的处理并没有走统一的线程池处理方案；

优化方案：
* 抽取公共代码，定义日志和业务日志抽象类，分别使用线程池进行处理；


<a name="4a1602db"></a>
#### GC分析：
对系统的GC日志进行了分析：<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/104361/1553510944011-8128b5a2-5a3e-4452-a559-a33651c2fde4.png#align=left&display=inline&height=494&name=image.png&originHeight=987&originWidth=2000&size=383029&status=done&width=1000)<br />发现频繁触发fgc，且fgc失败，old区 非常大，gc无法回收，怀疑存在大对象导致的内存无法回收。<br /><br />通过jmap查看进程占用的大对象；<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/104361/1553521339182-edde4ea1-1868-4c42-b641-da11d0b9341c.png#align=left&display=inline&height=368&name=image.png&originHeight=736&originWidth=1480&size=589499&status=done&width=740)<br /><br />发现系统内存中ots对象数据占用超过3G；与OTS同学沟通后，发现是client的使用问题。

<a name="9a39e603"></a>
#### 问题解决：
OTS写入优化，改用tablestoreWrite代替AyncClient，增加线程数控制、内存buffer控制等；
1. AyncClient处理时，当线程池满之后，会把新进来的数据全部堆在内存中，会导致内存占满；
1. tablestoreWrite采用了RingBuffer无锁写入，当线程池满且队列满之后，会阻塞调用线程；


<a name="216c0e94"></a>
#### 新问题出现：
系统灰度上线后，OTS写数据出现hang住；出现数据洪峰，导致处理线程block；<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/104361/1553521398649-dfe5511c-8d94-4f44-a50f-c15f86e35fb5.png#align=left&display=inline&height=265&name=image.png&originHeight=530&originWidth=1538&size=378791&status=done&width=769)

此时 jstack数据如下：<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/104361/1553076983556-e1aa1249-0711-4f7c-9986-27dc6a52cf54.png#align=left&display=inline&height=346&name=image.png&originHeight=692&originWidth=1284&size=281191&status=done&width=642)

![image.png](https://cdn.nlark.com/yuque/0/2019/png/104361/1553521506624-1e2cd35c-105f-4237-b2f0-9aff7fdab0a0.png#align=left&display=inline&height=252&name=image.png&originHeight=504&originWidth=1476&size=492669&status=done&width=738)

解决方案：
* 拆分线程池； 不能因为写监控数据而导致进程数据无法写入；进程的数据量与监控指标的数据量 不再一个数量级上。
* 针对ots数据写入增加限流保护，并进行block数据统计；
* 进一步定位导致TableStoreWriter写数据卡主的问题；


对系统进行offline，观察线程池队列处于满状态，预期是不再有新流量进来时，线程池队列数据会被消费，但是观察了二十分钟后，线程池队列数据依然处于慢状态，说明线程hang住了。


升级到OTS新版本后，持续观察，一段时间后，问题继续浮现。在仔细review代码，发现消费RingBuffer的线程池的拒绝策略采用了DiscardPolicy丢弃，也就是当系统处理不过来时进行丢弃，但是请求丢弃了会导致外层的线程hang住；<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/104361/1553154377682-13135f96-d88e-49fb-8eb6-07508cc6559f.png#align=left&display=inline&height=136&name=image.png&originHeight=271&originWidth=1558&size=336841&status=done&width=779)<br />具体为什么这里的线程池采用discard策略后会导致线程hang住，OTS同学给出的解释是“RingBuffer的数据会丢到线程池里处理，通过Semaphore来协同，如果executor丢弃了请求，那外层就会hang住”，此设计缺失不优雅，算是OTS Client的一个bug。

<a name="83616992"></a>
#### 最终解决：
* 修改线程池的拒绝策略后，灰度部分机器持续观察（上一次线上出现问题的时间周期大约为1小时，并且在一个波峰请求之后）。
* 线上丢弃的日志量恢复为0，且写ots正常。

<a name="8c9d507e"></a>
### 总结：
我们时刻需要对线上系统的容量和现状有清晰的了解，知道系统内存当前的水位，内存中那些线程在运行，什么数据对象会占用大量内存，系统还能接受多大的量，系统的极限负载是多少等。其次，对于线上稳定性问题定位，我们还需要熟练使用一套排查工具，辅助定位问题。

<a name="1917fd08"></a>
### 常用命令：
top ； # 查看系统CPU、Mem占用；<br />top -Hp pid； #查看具体进程下线程的cpu占用统计， pid到nid转换：printf '0x%x'；<br />iostat -d -k 1 10； # 统计磁盘IO；<br />tsar --cpu  --mem --traffic  --io ； #利用tsar来统计各项监控指标；<br />jmap -histo:live pid | head -20  #查看进程占用内存前20个对象；<br />jstack pid > jstack.log # 打印对应进程线程栈信息，常配合fastthread.io使用；<br />jstat –gcutil [pid] 1000 10 # 查看进程gc信息；


<a name="1b3b6b96"></a>
### 常用工具：
[https://gceasy.io/](https://gceasy.io/)         用于分析gc日志；<br />[https://fastthread.io/](https://fastthread.io/)    用于分析线程栈；<br />[https://github.com/alibaba/arthas](https://github.com/alibaba/arthas)  Arthas;


