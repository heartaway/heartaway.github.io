---
layout: post
title: Spring中property-placeholder的使用与解析
categories: Java
description:
keywords: spring-property,placeholder,占位符
date: 2018-01-19
---
在我们程序开发中，进程会需要把一些变量通过property方式进行提取，方便不同环境配置不同的属性，替换变量的方法通常有两种，一种是静态替换，一种是动态替换；所谓静态替换，是在打包编译的时候，把变量替换掉，动态替换，是在程序运行起来时，通过把属性注入到程序的环境变量中，类初始化的时候，再使用环境变量进行替换的一种方法。

静态替换常用工具：autoconfig
动态替换常用工具：spring.property-placeholder

## spring动态替换变量实践

简洁配置法：

```java
<context:property-placeholder location="classpath:xxx.properties" 
ignore-unresolvable="true"/>
```
等价于：

```java
    <bean id="propertyConfigurer" class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
        <property name="location">
            <value>myConfig.properties</value>
        </property>
    </bean>
```

完整配置属性：

```java
<context:property-placeholder   
        location=""  
        file-encoding=""  
        ignore-resource-not-found=""  
        ignore-unresolvable=""  
        properties-ref=""  
        local-override=""  
        system-properties-mode=""  
        order=""  
/> 
```
ignore-resource-not-found：如果属性文件找不到，是否忽略，默认false，即不忽略，找不到将抛出异常 

ignore-unresolvable：是否忽略解析不到的属性，如果不忽略，找不到将抛出异常 

order：当配置多个<context:property-placeholder/>时的查找顺序

不推荐将ignore-resource-not-found和ignore-unresolvable的值设置为ture，默认为false，可以有效避免程序运行异常

#### 使用PropertySource注解配置
Spring3.1添加了@PropertySource注解,方便添加property文件到环境.

#### properties的注入与使用
* java中使用@Value注解获取

```java
@Value( "${jdbc.url}" )
private String jdbcUrl;
```

* 在Spring的xml配置文件中获取

```java
<bean id="dataSource">
  <property name="url" value="${jdbc.url}" />
</bean>
```

#### 配置多个property-placeholder属性

Spring容器是采用反射扫描的发现机制，通过标签的命名空间实例化实例，当Spring探测到容器中有一个org.springframework.beans.factory.config.PropertyPlaceholderCVonfigurer的Bean就会停止对剩余PropertyPlaceholderConfigurer的扫描，即只能存在一个实例！

所以一遍不建议配置多个property-placeholder对象，但是在必须使用多个的场景下，如何配置呢？

```java
<context:property-placeholder location="xxx.properties" ignore-unresolvable="true"/>

<context:property-placeholder location="xxx.properties" ignore-unresolvable="true"/>
```
需要设置ignore-unresolvable="true"，否则后面的property-placeholder不会被加载；


ignore-unresolvable单独使用来看是“是否忽视不存在的配置项”，不仅如此，其还有一个隐含意思：是否还要扫描其他配置项：如果为false，则会忽视后续的property-placeholder，如果需要配置多个property-placeholder则应该设置为true；

#### context:property-placeholder 工作原理
在 ContextNamespaceHandler 中对于 context中的property-placeholder 标签，会采用PropertyPlaceholderBeanDefinitionParser解析器进行解析；

```java
public class ContextNamespaceHandler extends NamespaceHandlerSupport {

	@Override
	public void init() {
		registerBeanDefinitionParser("property-placeholder", new PropertyPlaceholderBeanDefinitionParser());
		registerBeanDefinitionParser("property-override", new PropertyOverrideBeanDefinitionParser());
		registerBeanDefinitionParser("annotation-config", new AnnotationConfigBeanDefinitionParser());
		registerBeanDefinitionParser("component-scan", new ComponentScanBeanDefinitionParser());
		registerBeanDefinitionParser("load-time-weaver", new LoadTimeWeaverBeanDefinitionParser());
		registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
		registerBeanDefinitionParser("mbean-export", new MBeanExportBeanDefinitionParser());
		registerBeanDefinitionParser("mbean-server", new MBeanServerBeanDefinitionParser());
	}

}
```
PropertyPlaceholderBeanDefinitionParser解析器会将property-placeholder 标签解析为一个PropertySourcesPlaceholderConfigurer的单例 bean 。

可以看出 PropertySourcesPlaceholderConfigurer 或者 PropertyPlaceholderConfigurer 仅仅是做了一个配置文件的解析工作，真正的注入并不由它们完成，而是托付给了Spring 的Bean初始化流程。 
之所以这么做可以生效，是因为这两个类实现了 BeanFactoryPostProcessor 接口，这个接口的优先级高于后续的Spring Bean。

属性元素的注入依赖于 AutowiredAnnotationBeanPostProcessor#postProcessPropertyValues。 通过解析PropertySourcesPlaceholderConfigurer 查询得到元素值。 

```java
    public PropertyValues postProcessPropertyValues(PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException {
        InjectionMetadata metadata = this.findAutowiringMetadata(beanName, bean.getClass());

        try {
            metadata.inject(bean, beanName, pvs);
            return pvs;
        } catch (Throwable var7) {
            throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", var7);
        }
    }
```


PropertySourcesPlaceholderConfigurer本质上是一个BeanFactoryPostProcessor。解析XML的流程在BeanFactoryPostProcessor之前， 优先将配置文件的路径以及名字通过Setter传入PropertySourcesPlaceholderConfigurer。

如上BeanFactoryPostProcessor的优先级又优于其余的Bean。因此可以实现在bean初始化之前的注入。

#### Spring @Value注入流程
1. Spring Context 的初始化开始
2. 读取到context:property-placeholder标签或者PropertySourcesPlaceholderConfigurer
3. 解析并实例化一个PropertySourcesPlaceholderConfigurer。同时向其中注入配置文件路径、名称
4. PropertySourcesPlaceholderConfigurer自身生成多个StringValueResolver备用，Bean准备完毕
5. Spring在初始化非BeanFactoryPostProcessor的Bean的时候，AutowiredAnnotationBeanPostProcessor 负责找到Bean内有@Value注解的Field或者Method
6. 通过PropertySourcesPlaceholderConfigurer寻找合适的StringValueResolver并解析得到val值。注入给@Value的Field或Method。(Method优先)2
7. Spring的其他流程。

参考：
http://blog.csdn.net/qyp199312/article/details/54313784



