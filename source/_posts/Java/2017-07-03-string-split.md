---
layout: post
title: Java split 引发的坑
categories: Java
description:
keywords: split
date: 2017-07-03
---

我们在代码编写中会进程需要使用数据截断处理，比如 30.23 我们期望获取整数部分，按照“.”进行分割；

可能的代码示例：

```java
   "30.23".split(".")[0]
```
如果我们这么写，那结果一定不是我们期望的，查看string 类中的split方法：
![](/images/posts/20170703/14990720990548.jpg)

我们看到，它接收的参数 **正则表达式**，恰好点是正则表达式的特殊字符，命中 

```
".$|()[{^?*+\\".indexOf(ch = regex.charAt(0)) 
```
条件，所以执行的实际逻辑为：

```java
Pattern.compile(regex).split(this, limit)
```
"." 表示匹配任何字符，那么按照任何字符进行分割，将得到""空字符串数组，Pattern.split在构造结果时，由于limit=0，会把结尾的空字符串丢弃，结果就返回了一个长度为0的String对象数组 new String[0] ,此时我们使用[0]获取数据中第一个元素时，会抛出数组越界的异常：

```java
Exception in thread "main" java.lang.ArrayIndexOutOfBoundsException: 0
```
如果我们期望使用string.split(regex)方法，对于特殊字符，需要进行转义处理，比如：

```java
   "30.23".split("\\.")[0]
```


常用的字符拆分方案有以下几种：

#### 方法二： org.apache.commons.lang.StringUtils.split 
此方法使用完整的字符串作为参数，而不是regex。底层调用splitWorker方法；

```java
        String number = "30.23";
        String[] split = StringUtils.split(number, ".");
        for (String s : split) {
            System.out.println(s);
        }
```
输出结果：

```java
30
23
```
### 方法三：com.google.common.base.Splitter
使用Google Guava包中提供的分割器splitter，它提供了更加丰富的分割结果处理的方法，比如对结果前后去除空格，去除空字符串等；

```java
String number = "30.23";
Iterable<String> split = Splitter.on('.').trimResults().omitEmptyStrings().split(number);
for (String s : split) {
     System.out.println(s);
}
```


