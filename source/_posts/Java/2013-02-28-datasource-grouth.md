---
layout: post
title: Java 数据库连接(dataSource)的演进
categories: Java
description:
keywords: datasource 数据库连接
date: 2013-02-28
---

#### 原生方法
* 加载JDBC 驱动：


```java
Class.forName(driver);
// mysql 数据库：“com.mysql.jdbc.Driver”
```

* 建立数据库连接：

```java
Connection conn = DriverManager.getConnection(url,userName,password);
```

*  创建 statement，用来执行SQL 语句


```java
Statement statement = conn.createStatement();
```

* 执行 SQL 语句：


```java
ResultSet rs = statement.executeQuery(sql);
```

* 关闭记录集，关闭声明，关闭连接对象

##### 不足：
1. 每次使用都要创建连接，使用完毕后还必须关闭连接，操作繁琐，易出错；
2. 连接数据库资源不便统一管理；

#### 使用Spring的 JDBC 方法：
* 引入 spring-jdbc.jar 包
* 添加 dataSource配置


```java
<bean id="dataSource">
    <property name="driverClassName" value="org.springframework.jdbc.datasource.DriverManagerDataSource">
    </property>
    <property name="url" value="jdbc:oracle:thin:@10.217.3.2:1521:orcl">
    </property>
    <property name="username" value="*"></property>
    <property name="password" value="*"></property>
</bean>
```

DriverManagerDataSource 类位于 org.springframework.jdbc.datasource 包下。 当然这里还可以选择 SingleConnectionDataSource
DriverManagerDataSource -> 在每一个连接请求时都新建一个连接；
SingleConnectionDataSource ->     在每一个连接请求时都返回同一个连接；

* 获取dataSource bean对象


```java
ApplicationContext ctx = new ClassPathXmlApplicationContext("applicationContext.xml");
ds = (DataSource)ctx.getBean("dataSource");
```

* 获取连接对象Connection和Statement


```java
Connection conn = ds.getConnection();
Statement sm = conn.createStatement();
```

* 执行向数据库插入记录操作


```java
String sqlString = "insert into bryanttesttable values(2,'bryant')";
sm.execute(sqlString);
```

##### 优势：
1. 更干净的 代码；
2. 更简单的使用；
3. 更好的异常与资源处理；

Spring JDBC 介绍：模版设计模式（核心包包含JdbcTemplate），Spring JDBC 异常处理 ；这些会在下一章节来具体介绍Spring JDBC的优雅设计 和 是如何在 原生JDBC 上做封装的。

#### 使用Spring的 数据库连接池 DBCP 方法：（四个流行的Java连接池）

需要引入commons-collections.jar、commons-dbcp.jar和commons-pool.jar。


```java
<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource">
<property name="driverClassName" value="oracle.jdbc.driver.OracleDriver"></property>
<property name="url" value="jdbc:oracle:thin:@192.168.1.35:1521:orcl"></property>
<property name="username" value="or_meal"></property>
<property name="password" value="or_meal"></property>
<property name="maxActive" value="100"></property>
<property name="maxIdle" value="30"></property>
<property name="maxWait" value="10"></property>
</bean>
```

#### 使用 JNDI 连接数据库

1、SpringJNDI数据源配置信息：


```java
<bean id="dataSource" class="org.springframework.jndi.JndiObjectFactoryBean">
   <property name="jndiName">
    <value>java:comp/env/jcptDataSourceJNDI</value>
   </property>
  </bean>
```

  jcptDataSourceJNDI是tomcat或者其他应用服务器配置的JNDI.

2、关于JNDI的配置(tomcat)：
  修改tomcat目录conf/context.xml文件：


```java
<Resource name="jcptDataSourceJNDI" auth="Container" type="javax.sql.DataSource"
      maxActive="100" maxIdle="30" maxWait="10"   username="tysp"
      password="12345678" driverClassName="oracle.jdbc.driver.OracleDriver"
      url="jdbc:oracle:thin:@192.168.1.35:1521:orcl"/>  
```

3、通过JNDI获取DataSource:


```java
<Resource name="jcptDataSourceJNDI" auth="Container" type="javax.sql.DataSource"
      maxActive="100" maxIdle="30" maxWait="10"   username="tysp"
      password="12345678" driverClassName="oracle.jdbc.driver.OracleDriver"
      url="jdbc:oracle:thin:@192.168.1.35:1521:orcl"/>  
```

