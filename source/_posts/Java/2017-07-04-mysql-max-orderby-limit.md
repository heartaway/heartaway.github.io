---
layout: post
title: Mysql MAX/MIN vs ORDER BY and LIMIT
categories: Java
description:
keywords: Max Order By Limit
date: 2017-07-04
---
我们在进行数据库的TOP N 操作时，又两种选择，一种使用内置的函数MAX/MIN，另外一种是采用 ORDER BY  和 LIMIT的组合，那么在什么情况下使用那种策略呢，哪种效率会更高一些呢？


在最坏的情况下，如果你正在寻找在一个无索引的字段，使用min()需要全表扫描。使用order by + limit 要求一个排序后的文件。如果碰到一个大表，性能差异会非常大。在我本机上测试了一个106000行的随机数据表，min()带消耗了0.36s,而order by + limit消耗了0.84s。如果对于一个索引列，两者的差别并不是太大。

**总的来说：**
似乎min()性能更高，它在最坏的情况下的速度更快，在最好的情况下，没有什么区别；

通过“show status like 'last_query_cost'” 查看执行成本；分别测试了在无索引列下使用max 和 order by limit 两种查询条件下的cbo，分别为 228322 和 67359，结果展示使用max的成本要比order by更高,这点还需要进一步探究；

参考：
https://stackoverflow.com/questions/426731/min-max-vs-order-by-and-limit


