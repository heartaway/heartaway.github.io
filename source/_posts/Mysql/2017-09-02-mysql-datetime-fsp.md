---
layout: post
title: Mysql时间精度丢失，导致秒被四舍五入
categories: Mysql
description:
keywords: mysql datetime 时间精度
date: 2017-09-02
---



### 问题描述：
Mysql中一张表时间类型采用的是dateTime格式，当后端通过Mybatis将new Date() 传递给数据库进行存储是，发现存入时间是**2016-12-23 16:41:55**，但是数据库中插入的时间却是**2016-12-23 16:41:5_6_*,导致采用同样的时间进行数据查询，却查不到数据。

### 问题排查：
1. 进行codeReview，未发现bug；
2. 本地将Mybatis的DEBUG日志打开，进行单步调试 
![](/images/posts/20170902/14824834942593.jpg)
![](/images/posts/20170902/14824835188662.jpg)
3. 日志输出如下：
![](/images/posts/20170902/14824828832109.jpg)

    ```java
    2016-12-23 16:43:26.960 [DEBUG]xxx.update [debug:145] ==>  Preparing: update xxx SET gmt_modified = now(), status = ?, collect_time = ? where id = ? 
    2016-12-23 16:43:26.964 [DEBUG][debug:145] ==> Parameters: SLEEPING(String), 2016-12-23 16:41:55.972(Timestamp), 9(Long)
    ```
4. 发现传入mybatis的date参数为携带毫秒数的时间，到服务端之后多了一秒，但是有时候又是相等的，初步怀疑是毫秒数会被四舍五入了，多次执行后，正式了这个观点；这也能解释为什么我们再以同样的Date参数为什么偶尔查不到数据的问题了。

### 根因定位：
1. 确认Mysql Server的版本信息：
![](/images/posts/20170902/14826733264139.jpg)

2. 确认Datetime的格式信息：
![](/images/posts/20170902/14826731196884.jpg)

3. JDBC（JDBC版本5.1.26）中判断是否MysqlServer（Server版本5.6.4）的版本是否支持毫秒：
com.mysql.jdbc.PreparedStatement#detectFractionalSecondsSupport:923L
![](/images/posts/20170902/14826735377431.jpg)

4. JDBC（JDBC版本5.1.26）中设置毫秒数函数：
com.mysql.jdbc.PreparedStatement#setTimestampInternal
首先会判断是否设置了useLegacyDatetimeCode=false，如果设置了，那么会走新方法newSetTimestampInternal：
![](/images/posts/20170902/14826767849619.jpg)
可以看到，新方法中会不管mysql服务端的版本信息是什么，默认都会把毫秒数携带上发送给服务端，让服务端来判断毫秒数怎么处理。当我们代码中把这部分携带了毫秒的时间发送给服务端时，由于服务端版本为5.6.16，所以不支持毫秒数，所以就被四舍五入进行存储到秒了。

    在看看如果设置了useLegacyDatetimeCode=true的逻辑：
![](/images/posts/20170902/14826767169568.jpg)
那useLegacyDatetimeCode到底是什么？
>Note that to get this scenario to work with MySQL (since it doesn't support per-value timezones), you need to configure your server (or session) to be in UTC, and tell the driver not to use the legacy date/time code by setting useLegacyDatetimeCode to "false". This will cause the driver to always convert to/from the server and client timezone consistently.

5. 精度问题：
Mysql中 DATE, DATETIME, and TIMESTAMP 三种类型的区别，可以参考：http://dev.mysql.com/doc/refman/5.6/en/datetime.html。

    需要纠正一个错误的观点，那就是有同学认为DATETIME比TIMESTAMP精度低，这个是错误的，他们两个能达到的精度是一致的，只不过，他们可表示的时间范围不同而已；
测试Mysql5.7.14版本下，三种时间类型的数据插入：
![](/images/posts/20170902/14826747022343.jpg)
对于一个立志存活102年的企业来说，选择DATETIME比TIMSTAMP是一个更明智的选择，可以表示的范围更大，而且不受系统时区的影响；
6. 总结：
至此，我们知道了三种时间类型在不同Mysql版本不同的所展示的不同的特点，以及不同Mysql版本对毫秒的支持情况；目前我们服务端由于采用的mysql版本是5.6.16(低于5.6.4)，所以还不支持毫秒数的存储，当客户端提交过来的小数部分超长时（客户端默认配置了useLegacyDatetimeCode=false），server会默认做四舍五入，不会出现任何的警告或异常。
那么在这种情况下，问题如何解决呢？
①客户端在进行时间存储的时候，主动通过一次SimpleDateFormat函数format和parse将毫秒数过滤掉，相当于重置为0；
②升级Mysql Server版本到5.6.4及以上；

### 扩展：Java中的Date类型：
java.util.Date 与 java.sql.Date 的关系？
java.util.Date 就是在除了SQL语句的情况下面使用；
java.sql.Date 继承自 java.util.Date 
java.sql.Date 是针对SQL语句使用的，它只包含日期而没有时间部分；



