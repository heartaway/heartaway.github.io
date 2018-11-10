---
layout: post
title: Mysql新增字段到大数据表导致锁表
categories: Mysql
description:
keywords: mysql lock ddl 锁表
date: 2016-06-15
---
生产环境对Mysql中一张表（sc_stockout_order）做新增一个字段操作：

```java
  ALTER TABLE `sc_stockout_order` ADD `route_remarks` VARCHAR(1024)  CHARACTER SET utf8mb4  NULL  DEFAULT NULL  COMMENT '物流备注'  AFTER `remarks`;
```

Mysql的配置：

![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/04f830403a41d77a161d11469427417f.png)

应用系统也开始报错：


```java
Cause: org.springframework.jdbc.CannotGetJdbcConnectionException: Could not get JDBC Connection; 
        nested exception is org.apache.tomcat.jdbc.pool.PoolExhaustedException: 
        [DubboServerHandler-10.162.99.129:20880-thread-105] Timeout: Pool empty. 
        Unable to fetch a connection in 50 seconds, none available[size:80; busy:79; idle:0; lastwait:50000].
        at org.apache.ibatis.exceptions.ExceptionFactory.wrapException(ExceptionFactory.java:26) ~[mybatis-3.2.8.jar:3.2.8
```

从日志上可以看出JDBC连接池拿不到可用连接了，连接池大小为80，当前空闲为0，由于有2太机器，所以这个时候总的最大连接数为160，虽然数据库的最大连接数是300，但是已经达到了客户端的配置；其实这里的配置不太可以，2太机器的话，可以配高一点，比如每台最大连接数可以配置为150。

发现问题后，首先为了解决问题，在Mysql客户端中  show processlist  发现由于ALTER TABLE xx 这条语句导致大量的Query条件的语句处于等待状态，为了减少继续产生问题，必须先解决问题，通过 kill processId  杀掉修改表结构的语句，发现马上processlist恢复正常。

show processlist 中还出现了一条语句：


```java
Waiting for table metadata lock
```

sc_stockout_order 中现有字段个数 91 个，数据量为 3959102  条,  数据量还是比较大的。

#### Mysql的连接数：

![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/62bb08da180f2be0647b257b8c9ea9bb.png)

从图中也可以看出活跃连接数达到160之后就不变了，顶端蓝色水平线部分，当杀掉阻塞进程后，连接得到释放。

#### 事后，查找资料，进行原因分析：
Mysql在5.6版本之前，直接修改表结构的过程中会锁表，具体的操作步骤如下：

1. 首先创建新的临时表，表结构通过命令ALTAR TABLE新定义的结构
2. 然后把原表中数据导入到临时表
3. 删除原表
4. 最后把临时表重命名为原来的表名

>Historically, many DDL operations on InnoDB tables were expensive. Many ALTER TABLE operations worked by creating a new, empty table defined with the requested table options and indexes, then copying the existing rows to the new table one-by-one, updating the indexes as the rows were inserted. After all rows from the original table were copied, the old table was dropped and the copy was renamed with the name of the original table.

但是在Mysql 5.6 之后引入了Online DDL：

>MySQL 5.6 enhances many other types of ALTER TABLE operations to avoid copying the table. Another enhancement allows SELECT queries and INSERT, UPDATE, and DELETE (DML) statements to proceed while the table is being altered. This combination of features is now known as online DDL.

那么Online DDL下是如何实现的呢？
参考：[http://www.cnblogs.com/cchust/p/4639397.html](http://www.cnblogs.com/cchust/p/4639397.html)

Mysql 5.6 虽然引入了Online DDL，但是并不是修改表结构的时候，一定不会导致锁表，在一些场景下还是会锁表的，比如

①某个慢SQL或者比较大的结果集的SQL在运行，执行ALTER TABLE时将会导致锁表发生；

②存在一个事务在操作表的时候，执行ALTER TABLE也会导致修改等待；

查看我们mysql的版本：


```sql
SELECT VERSION();
5.6.16-log
```

我们通过Mysql的慢SQL控制台，也在发生问题的时间段内没有出现慢SQL，所以需要排除第一种可能性；
由于当时没有保留现场，所以当时是不是由于事物导致的锁表，现在也无从查起，这只能下次查看分析了。


#### 总结：
对大表的修改以后千万要注意，修改需要满足一下条件：

* 需要找一个流量比较小的时候；
* 需要查看当前是否有未提交的实物；
* 需要测试环境预演练，有一定预期；
* 知晓表数据大小（SELECT  DATA_LENGTH + INDEX_LENGTH  FROM TABLES WHERE TABLE_SCHEMA=’logistics’ AND TABLE_NAME=’stockout_order’ ;）


Mysql大表一般DBA都会做归档操作，把大表转为小表；或者使用 [percona-toolkit ](https://www.percona.com/doc/percona-toolkit/2.2/pt-online-schema-change.html)工具来辅助完成DDL操作，关于PT的工具原理和介绍：[http://www.cnblogs.com/zhoujinyi/p/3491059.html](http://www.cnblogs.com/zhoujinyi/p/3491059.html)
有关 Metadata lock wait  请参考：[https://help.aliyun.com/knowledge_detail/41723.html](https://help.aliyun.com/knowledge_detail/41723.html)

