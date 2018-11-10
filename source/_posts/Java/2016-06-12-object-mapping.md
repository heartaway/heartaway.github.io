---
layout: post
title: Object Mapping in Java
categories: Java
description:
keywords: mapping
date: 2016-06-12
---
我们在Java代码编写中经常会遇到DO 、DTO之间的对象隐射转换，我们在设计DO、DTO的时候一般会尽量让对象名称、对象属性保持一致，利于属性拷贝，但是现实场景中可能存在一些对象名称不一致、对象类型不一致的情况，不同的拷贝方案，性能与使用场景也可能存在不一样，那么在众多的对象拷贝框架中如何选择合适的使用呢？

常用对象属性拷贝方法：

1. commons-beanutils 框架中的 BeanUtils
2. commons-beanutils 框架中的 PropertyUtils
3. ModelMapper
4. Cglib中提供的BeanCopier, BulkBean,BeanMap,FastClass/FastMethod
5. Orika
6. Dozer
7. HardCopy，手工硬编码

有人做了一张性能对比图：

![](http:/ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/7236b8abc31ef20753e7d002106a0caf.png)

![](/images/posts/20160612/object-mapping-per-hightchart-two.png)
#### HardCopy：
手工硬编码的方式是基本是效率最高的方式，但是当一个类有几十个属性的时候，代码编写效率低下，而且丑陋，最重要的是，当新扩展一个字段后，往往容易忽略在mapping convert文件中添加相应的属性隐射，给业务带来一定的潜在风险（error-prone）。硬编码并不是一无是处，当对象属性比较固定，且对性能要求非常高的时候，硬编码是最好的选择方案。但是很多Object的 大量属性需要编写Convert方法时，就感觉很浪费时间，有没有更有效率的编写方法呢？当然，我自己写了一个Convert方法生成器（密码: ycag，Console输出，需要复制粘贴），这样就可以大量节省硬编码ModelConvert的时间。
#### BeanUtils：
因为我们项目中最常用的就是Spring-bean，所以最先想到使用属性拷贝的就是Apache的BeanUtils.copyProperties方法。

##### 工作原理：
BeanUtils通过反射机制的方式从orig中获取属性信息和属性的可读、可写方法（通过get、set方法，如果属性没有get、set方法，则此属性不会被拷贝），调用orig的get方法获取属性值，将属性值写入dest的对应属性中（使用dest的set方法）。可以看出BeanUtils是严格要求orig、dest必须符合bean规范，有get、set方法，无法完成不同属性名之间的隐射拷贝。

##### 对象类型转换：
BeanUtils为了方便对象类型转换，还提供了ConvertUtilsBean类，此类可以注入一系列Converter对象，提供对象的初始值或者其他转换方法，比如设置Integer的默认值为0 或 从Date 类型到long类型，我们在使用过程中一般NULL与0是不一样的，所以不期望对象转换过程中默认设置值，所以会自定义对象类型转换器。由此说明BeanUtils是对属性类型没有强一致性判断的。

![](/images/posts/20160612/bean-util-sample.png)

对象拷贝深度：待测试。
由于大量采用反射机制&严格的参数校验，所以性能较差，不建议使用。
#### PropertyUtils：
PropertyUtils实际上使用的也是beanUtils，但是在初始化的时候就会发现，BeanUtils初始化了ConvertUtilsBean和PropertyUtilsBean，而PropertyUtils只初始化了PropertyUtilsBean，说明PropertyUtils不支持属性类型自动转换的功能(如果类型不同则会抛出异常)，而BeanUtils支持属性类型自动转换的功能，这也是两者的区别。

性能上，PropertyUtils 与 BeanUtils稍微好一点。

#### ModelMapper：
ModelMapper能用更加紧凑的代码对Java对象进行映射，在更简单的情况下甚至可以实现零配置。它支持以下特性：

* 基于名称的对象属性映射
* 复制公开的、受保护的和私有的字段
* 略过某些字段
* 可用转换器来影响映射（如将字符串转换为小写）
* 在不同类型的字段间进行映射（如将字符串转换为数字）
* 采用不同的条件进行映射
* 默认条件不充分时采用松散的映射策略
* 对映射过程进行验证以确保所有字段都被处理
* 对特殊情况下的映射过程进行完全可定制化的控制
* 与Guice或Spring集成

#### Cglib：BeanCopier
参考：[http://agapple.iteye.com/blog/799827](http://agapple.iteye.com/blog/799827)，据说性能比BeanUtils优一个数量级以上，但是其提供的功能也比较简单。
#### [Orika](http://orika-mapper.github.io/orika-docs/)：
Github中的源码：[https://github.com/orika-mapper/orika](https://github.com/orika-mapper/orika)
基于生成字节码的方式进行属性映射，是目前出了硬编码外性能最好的一款Object mapping工具，他比ModelMapper占用的内存会稍微多一点，空间换时间，这也符合计算机规律。不足时文档较少，API看起来有一些混乱，支持的特性不较少，其次一些属性的注册使用的是“弱类型”，当属性名称变更时容易被忽略（error-prone）。
#### [Dozer](http://dozer.sourceforge.net/)：
虽然Dozer提供了丰富的特性支持，但是由于其性能较差，所以这里就不做介绍。
除了以上介绍的几种之外还有一些其他的Object mapping工具类，比如 mapping4java、GeDA等。
#### 总结：
根据自己的需求选择一款合适且性能优雅的框架，如果觉得没有合适的，当然在时间和精力允许的情况下自己也可以去造一个轮子。

#### 参考：
[https://prezi.com/q7pd1ad2spro/copy-of-object-mapping-in-java/](https://prezi.com/q7pd1ad2spro/copy-of-object-mapping-in-java/)

[http://stackoverflow.com/questions/1432764/any-tool-for-java-object-to-object-mapping](http://stackoverflow.com/questions/1432764/any-tool-for-java-object-to-object-mapping)

[http://agapple.iteye.com/blog/1075671](http://agapple.iteye.com/blog/1075671)


