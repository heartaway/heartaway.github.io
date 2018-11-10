---
layout: post
title: 《Spring 技术内幕》之 bean的初始化
categories: Spring
description:
keywords: spring,bean
date: 2016-05-10
---

**BeanFactory：**指定IOC容器的基本功能规范； IOC是水桶，那么这个类就定义了水桶的基本功能;

**BeanDefinition：**对依赖反转模式中管理的对象依赖关系的抽象；它就像容器里装的水，有了这些基本数据，容器才能发挥作用；

## BeanFactory 与 FactoryBean的区别：

BeanFactory 是 Factory，也就是IoC 容器或对象的工厂，所有的Bean都是由BeanFactory 来管理的。

FactoryBean 是 bean，但不是一个简单的bean，而是一个能生产或修饰对象生成的工厂bean（工厂模式类似）；

我们可以认为直接的BeanFactory 实现是IoC容器的基本实现，而各种ApplicationContext的实现是IoC容器的高级表现形式。


IoC容器的建立过程:（XmlBeanFactory）

1. 创建IoＣ配置文件的抽象资源，这个抽象资源包含了BeanDefinition 的定义信息
2. 创建BeanFactory（实现类）
3. 创建一个载入BeanDefinition的读取器，然后通过一个回调配置给BeanFactory
4. 从定义好的资源位置读入配置信息。DefaultListableBeanFactory 非常重要的BeanFactory。

IoC容器 的初始化包括 BeanDefinition的Resouce定位、载入和注册这三个基本过程。ResourceLoader，BeanDefinition.
三大过程：

① BeanDefinition 的载入和解析
② BeanDefinition向 IoC容器中进行注册
③ IoC 容器的 依赖注入(bean 在使用时才进行初始化)

![](/images/posts/20160510/bean-init-three-step.png)

整个过程可以理解为是容器的初始化过程。第一个过程是 ApplicationContext 的职责范围，第二步是 BeanFactory的职责范围。


Resource 定义了资源文件

ResourceLoader  对Resource进行加载


```java

ResourceLoader
  |__ DefaultResourceLoader
       .    |__  ServletContextResourceLoader/ServletContextResourceLoader/FileSystemResourceLoader
       .         AbstractApplicationContext
       .                  |__ AbstractRefreshableConfigApplicationContext
       .                          |__  AbstractXmlApplicationContext
       .                                     |__ 各种 XmlAplicationContext
       .                                AbstractRefreshableWebApplicationContext
       ResourcePatternResolver
```

Cotext 中的构造方法中的 refresh() 代表启动容器，是bean加载和解析的入口。


BeanDefinition  定义了 bean的数据结构

BeanDefinitionReader 是对Bean的加载；


```java

BeanDefinitionReader 
 |__ AbstractBeanDefinitionReader
      |__ PropertiesBeanDefinitionReader   
           XmlBeanDefinitionReader
```


两大容器体系： BeanFactory 和 ApplicationContext

![](/images/posts/20160510/beanFactory-applicationContext-hex.png)

左边黄色部分是 ApplicationContext 体系继承结构，右边是 BeanFactory 的结构体系,两个结构是典型模板方法设计模式的使用。

从该继承体系可以看出：

1.       BeanFactory 是一个 bean 工厂的最基本定义，里面包含了一个 bean 工厂的几个最基本的方法，getBean(…) 、 containsBean(…) 等 ,是一个很纯粹的bean工厂，不关注资源、资源位置、事件等。ApplicationContext 是一个容器的最基本接口定义，它继承了 BeanFactory, 拥有工厂的基本方法。同时继承了ApplicationEventPublisher 、 MessageSource 、 ResourcePatternResolver 等接口，使其 定义了一些额外的功能，如资源、事件等这些额外的功能。
2.      AbstractBeanFactory 和 AbstractAutowireCapableBeanFactory 是两个模板抽象工厂类。AbstractBeanFactory 提供了 bean 工厂的抽象基类，同时提供了 ConfigurableBeanFactory 的完整实现。AbstractAutowireCapableBeanFactory 是继承了 AbstractBeanFactory 的抽象工厂，里面提供了 bean 创建的支持，包括 bean 的创建、依赖注入、检查等等功能，是一个核心的 bean 工厂基类。
3.       ClassPathXmlApplicationContext之 所以拥有 bean 工厂的功能是通过持有一个真正的 bean 工厂DefaultListableBeanFactory 的实例，并通过 代理 该工厂完成。
4.       ClassPathXmlApplicationContext 的初始化过程是对本身容器的初始化同时也是对其持有的DefaultListableBeanFactory 的初始化。

BeanDefinition 的载入包括两部分，首先是通过调用XML 的解析器得到document 对象，但这些document对象 并没有按照Spring 的bean 规则进行解析，在完成通用的XML 解析以后，才是按照Spring 的bean 规则 进行解析的地方，按照Spring的bean 规则进行解析的过程是在 documentReader 中实现的。

具体容器初始化过程如下：

![](/images/posts/20160510/ioc-context-init.png)

依赖注入是发生在容器中的BeanDefinition 数据已经建立好的前提下进行的。


那么 IoC容器的依赖注入是怎么实现的 ？（getBean()的过程）

在bean 的创建和 对象依赖注入的过程中，需要根据BeanDefinition 中的信息来递归地完成依赖注入。


1. bean 是在什么时候被创建的，有哪些规则？
容器初始化的时候会预先对单例和非延迟加载的对象进行预先初始化(调用 getBean()方法)。其他的都是延迟加载是在第一次调用getBean 的时候被创建。从 DefaultListableBeanFactory 的 preInstantiateSingletons 里可以看到这个规则的实现。


2. Bean 的创建过程？
对于 bean 的创建过程其实都是通过调用工厂的 getBean 方法来完成的。这里面将会完成对构造函数的选择、依赖注入等


GetBean 的大概过程：

1. 先试着从单例缓存对象里获取。
2. 从父容器里取定义，有则由父容器创建。
3. 如果是单例，则走单例对象的创建过程：在 spring 容器里单例对象和非单例对象的创建过程是一样的。都会调用父类 AbstractAutowireCapableBeanFactory 的 createBean 方法。 不同的是单例对象只创建一次并且需要缓存起来。 DefaultListableBeanFactory 的父类 DefaultSingletonBeanRegistry 提供了对单例对象缓存等支持工作。所以是单例对象的话会调用 DefaultSingletonBeanRegistry 的 getSingleton 方法，它会间接调用AbstractAutowireCapableBeanFactory 的 createBean 方法。如果是 Prototype 多例则直接调用父类 AbstractAutowireCapableBeanFactory 的 createBean 方法。


doCreateBean 的流程：

1. 会创建一个 BeanWrapper 对象 用于存放实例化对象。
2. 如果没有指定构造函数，会通过反射拿到一个默认的构造函数对象，并赋予beanDefinition.resolvedConstructorOrFactoryMethod 。
3. 调用 spring 的 BeanUtils 的 instantiateClass 方法，通过反射创建对象。
4. applyMergedBeanDefinitionPostProcessors
5. populateBean(beanName, mbd, instanceWrapper); 根据注入方式进行注入。根据是否有依赖检查进行依赖检查。
6. initializeBean(beanName, exposedObject, mbd);判断是否实现了 BeanNameAware 、 BeanClassLoaderAware 等 spring 提供的接口，如果实现了，进行默认的注入。同时判断是否实现了 InitializingBean 接口，如果是的话，调用 afterPropertySet 方法。

