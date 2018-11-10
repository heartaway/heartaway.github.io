---
layout: post
title: 计算Java方法执行时间
categories: Java
date: 2016-05-31
---

我们在对系统进行性能调优的时候，很多时候都会统计接口的调用时间或者内部方法的执行时间来查看具体的性能瓶颈点。

## 编码式
一般比较简单的左右是在方法开始和结束的时候分别添加时间记录方法：

```java
long startTimeMillis = System.currentTimeMillis()
…..
long endTimeMillis = System.currentTimeMillis()
long spendTime = endTimeMillis - startTimeMillis;
```

## StopWatch
当然你可以可以通过Apache-Commons工具包中的工具类：


```java
org.apache.commons.lang3.time.StopWatch
```

StopWatch 底层也是采用的System.currentTimeMillis()进行时间统计，使用StopWatch的优点：

* 封装好了基本的时间统计方法，你直接在方法开始和结束的时候调用一次就好了，使用简单；
* 他还提供了时间统计挂起、恢复和复位等功能，这点通过方法一是无法实现的；

使用StopWatch的弊端：

* 会抛出IllegalStateException或者RuntimeException异常，这点使用者如果不进行封装容易引起业务异常。一般我们不期望一个测试性能的代码影响到了正常的业务逻辑；

## 使用Spring AOP方式：

示例如下：


```java
public class MethodTimeActive implements MethodInterceptor {
 
    private static final Logger logger = LoggerFactory.getLogger(MethodTimeActive.class);
 
    /**
     * 自定义map集合，key：方法名，value：[0：运行次数，1：总时间]
     */
    public static Map<String, Long[]> methodTest = new HashMap<String, Long[]>();
 
 
    @Override
    public Object invoke(MethodInvocation methodInvocation) throws Throwable {
        StopWatch watch = new StopWatch();
        watch.start();
        Object object = methodInvocation.proceed();
        watch.stop();
        String methodName = methodInvocation.getMethod().getName();
        Long time = watch.getTime();
        if (methodTest.containsKey(methodName)) {
            Long[] x = methodTest.get(methodName);
            x[0]++;
            x[1] += time;
            logger.info("MethodTime  _method=" + methodName + " _totalCount=" + x[0] + " _totalTime=" + x[1] + " ms  _avgTime=" + x[1] / x[0] + "ms");
        } else {
            methodTest.put(methodName, new Long[]{1L, time});
            logger.info("MethodTime  _method=" + methodName + " _totalCount=1 _totalTime=" + time + "ms _avgTime=" + time + "ms");
        }
 
        return object;
    }
}
```

定义方法拦截器

spring-context 中对待监控的服务和方法进行配置（通过AOP的代理bean代理需要被检测的bean对象）：


```java
<bean id="venderSpuService" class="org.springframework.aop.framework.ProxyFactoryBean">
        <property name="target" ref="venderSpuServiceImpl"/>
        <property name="interceptorNames">
            <list>
                <value>spuServiceAdvisor</value>
            </list>
        </property>
    </bean>
 
    <bean id="methodTimeActive" class="com.sfebiz.vender.util.MethodTimeActive"/>
 
    <bean id="spuServiceAdvisor" class="org.springframework.aop.support.NameMatchMethodPointcutAdvisor">
        <property name="mappedNames">
            <list>
                <value>querySpuByProvider4Page</value>
                <value>querySpuByVSpuId</value>
                <value>stashSpu</value>
                <value>commitSpu</value>
            </list>
        </property>
        <property name="advice" ref="methodTimeActive"/>
    </bean>
```

需要maven依赖：


```java
            <dependency>
                <groupId>org.aspectj</groupId>
                <artifactId>aspectjweaver</artifactId>
                <version>1.8.6</version>
            </dependency>
```

遇到的问题：

* 通过AOP统计到的方法耗时，发现每次API调用打印了次接口调用，具体为什么还没有时间去排查。

## 其它：
1. 如果裸露的使用System.currentTimeMillis()在高并发场景下也会带来一定的耗时，因为每一个请求都会进行一次系统交互，可以通过内存缓存的方式定时更新当前时间，这样虽然性能提高了，但是可能导致统计的时间不够准确。
2. 如果期望使用System.currentTimeMillis()获取高精度的时间，可能需要注意了，因为System.currentTimeMillis()调用的还native方法，有依赖于操作系统。
3. 使用AOP代理服务于原始服务的性能差异还需要进行测试，所以目前还不太敢在生产环境上使用。

