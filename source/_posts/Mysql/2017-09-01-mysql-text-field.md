---
layout: post
title: How to select text data type for mysql
categories: Mysql
description:
keywords: mysql text
date: 2017-09-01
---

在进行后台页面模块配置化时，需要把前段页面的部分代码保存到db中，之前mysql的数据类型使用的是text，并没有考虑到text的长度限制，认为一般不会超过此限制，但是在运行中发现其中一个复杂的页面模块在保存时报错；

```sql
Caused by: com.mysql.jdbc.MysqlDataTruncation: Data truncation: Data too long for column 'xxx' at row 1
```

此时发现 xxx 字段的长度为text类型，需要保存的文本长度为 `65581`，mysql提供了四种类型的text类型，如下：


|Type | Maximum length|
|---|---|
|  TINYTEXT |           255 (2 8−1) bytes|
|      TEXT |        65,535 (216−1) bytes = 64 KiB|
|MEDIUMTEXT |    16,777,215 (224−1) bytes = 16 MiB|
|  LONGTEXT | 4,294,967,295 (232−1) bytes =  4 GiB|

我们存储的数据超过了text的存储长度（65535），所以修改数据类型为MEDIUMTEXT。

Mysql 中char、varchar、和text定义的空间大小和实际占用的关系又是怎么样的呢？

##### 基本知识：
1.  char（n）和varchar（n）中括号中n代表字符的个数，并不代表字节个数，所以当使用了中文的时候(UTF8)意味着可以插入m个中文，但是实际会占用m*3个字节。
2. 同时char和varchar最大的区别就在于char不管实际value都会占用n个字符的空间，而varchar只会占用实际字符应该占用的空间+1，并且实际空间+1<=n。
3. 超过char和varchar的n设置后，字符串会被截断。
4. char的上限为255字节，varchar的上限65535字节，text的上限为65535。
5. char在存储的时候会截断尾部的空格，varchar和text不会。
6. varchar会使用1-3个字节来存储长度，text不会。


| Value | CHAR(4) | Storage Required | VARCHAR(4) | Storage Required |
| --- | --- | --- | --- | --- |
| '' | '    ' | 4 bytes  | '' | 1 bytes |
| 'ab' | 'ab  ' | 4 bytes  | 'ab' | 3 bytes |
| 'abcd' | 'abcd' | 4 bytes  | 'abcd' | 5 bytes |
| 'abcdefgh' | 'abcd' | 4 bytes  | 'abcd' | 5 bytes |

#### 总体来说：
1. char，存定长，速度快，存在空间浪费的可能，会处理尾部空格，上限255。
2. varchar，存变长，速度慢，不存在空间浪费，不处理尾部空格，上限65535，但是有存储长度实际65532最大可用。
3. text，存变长大数据，速度慢，不存在空间浪费，不处理尾部空格，上限65535，会用额外空间存放数据长度，顾可以全部使用65535。
4. 当字符长度超过255的长度之后，使用varchar和text没有本质区别，只需要考虑一下两个类型的特性即可。


