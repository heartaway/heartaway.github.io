---
layout: post
title: 为什么SimpleDateFormat 不是线程安全
categories: Java
description:
keywords: simpleDateFormat 线程安全
date: 2016-03-10
---
查看SimpleDateFormat类的说明，在最后一条已经明确说明了此类非线程安全，建议为每一个线程创建一个单独的实例，如果多线程共享实例，务必保证同步性；

![](/images/posts/20160310/7BCC9F9B-0535-48BC-B247-3A13DFC9C60E.png)

要知道为什么不是线程安全的，首先我们需要知道SimpleDateFormat的作用和工作原理。

SimpleDateFormat继承了DateFormat,在DateFormat中定义了一个protected属性的 Calendar类的对象：calendar。只是因为Calendar类的概念复杂，牵扯到时区与本地化等等，Jdk的实现中使用了成员变量来传递参数，这就造成在多线程的时候会出现错误。

SimpleDateFormat最重要的两个方法，一个是format，一个是parse。

#### format方法：

![](/images/posts/20160310/simpledataformat-format.png)

可以看到，方法一开始就使用传入的date参数对属性calendar进行了初始化，试想如果此类是共享的，那么其它类还在使用此calendar的引用时，可能下一次获取到的值就与之前获取到的值不一样了。那format方法为什么要使用传入参数date对calendar的进行设置呢？为什么不可以把calendar作为方法参数在内部方法中传递呢？如果这里calendar是内部方法属性，并在subFormat中通过方法传递，一切问题就都解决了。作者这么做也许只是想减少一个参数的传递吧，但是这带了了诸多的线程安全问题。

#### parse方法：
首先从方法的注解上可以了解到parse的实现方式是通过对时间字符串逐步进行解析，然后通过java.text.CalendarBuilder.establish和calendar构建时间对象。 Date字符串解析中使用了calendar对象，在解析前，会调用Calendar#clear()方法，对引用的calendar对象的各个域进行初始化为默认值；然后把每次解析出来的时间片段设置到calendar相应的属性中，最后返回calendar中的time。

![](/images/posts/20160310/5C6ED2A3-C1DF-4937-8C59-A03C73582185.png)

不难想出在高并发情况下，如果calendar是多线程共享的，一定会出现A线程中calendar进行了clear(),导致B线程受到影响。

#### 无状态性：
思考：SimpleDateFormat类中是什么因素导致了此类不是线程安全的呢？是对calendar的引用，方法中对calendar的引用对象进行了修改，从而导致了此类是有状态的，我们一般在设计代码的时候尽量让自己的代码处于无状态，好处是，在各种情况下，无论并发掉多少次，结果总是一样的。

#### SimpleDateFormat线程安全的解决方案：
1. SimpleDateFormat设计为方法内实例；好处，从共享变为非共享，线程安全。弊端是每次创建性能开销大。
2. 在进行parse或者format时对SimpleDateFormat对象进行加锁；
3. 使用ThreadLocal，为每一个线程维护一个实例对象。
4. 使用Apache commons包中的FastDateFormat 或者 Joda-Time类库。

下午在代码开发中使用到的时间Util，采用的是方法三：


```java
/**
 * <p>
 * 由于SimpleDateFormat不是线程安全的，所以在作为静态工具类使用的时候需要特殊处理
 * </p>
 * Date: 16/3/10
 * Time: 下午3:38
 */
public final class DateUtil {

    private static final Logger logger = LoggerFactory.getLogger(DateUtil.class);
    public static final String DEF_PATTERN = "yyyy-MM-dd HH:mm:ss";//默认时间格式
    public static final String DATE_PATTERN = "yyyy-MM-dd";//日期格式
    //注意，这里初始化会有问题，后面会讲到（自己使用时需要删除此行）
    public static final Date START_NULL_DATE = parseDate("1970-01-01 08:00:00", DATE_PATTERN);

    private static final ThreadLocal<Map<String/**pattern**/, DateFormat>> threadLocal = new ThreadLocal<Map<String, DateFormat>>() {
        @Override
        protected Map<String, DateFormat> initialValue() {
            return new HashMap<String, DateFormat>();
        }
    };


    /**
     * 获取DateFormat
     *
     * @param pattern
     * @return
     */
    private static DateFormat getDateFormat(String pattern) {
        DateFormat dateFormat = threadLocal.get().get(pattern);
        if (dateFormat == null) {
            dateFormat = new SimpleDateFormat(pattern);
            threadLocal.get().put(pattern, dateFormat);
        }

        return dateFormat;
    }

}
```

在生产环境中使用过程中，Runtime期间JVM报错：


```java
java.lang.NoClassDefFoundError: Could not initialize class com.xxxx.vender.util.DateUtil
```

开始怀疑是因为把ThreadLocal作为静态初始化块导致的问题，但是查阅ThreadLocal的官方文档，给出的实例就是作为static属性使用，有同学说最好把DateUtil工具类做成单例模式进行使用，但是这个并没有找到报错的原因，后来有再本地进行了测试，发现抛出了NPE的异常，再仔细看看代码，发现有同学在ThreadLocal的初始化块之上添加了静态属性START_NULL_DATE,而START_NULL_DATE的初始化依赖threadLocal属性的初始化，抛出NPE是合乎情理的，那为什么生产环境上却出现的是java.lang.NoClassDefFoundError而不是NPE呢？

 

#### java.lang.NoClassDefFoundError到底是什么：
NoClassDefFoundError发生在JVM在动态运行时，根据你提供的类名，在classpath中找到对应的类进行加载，但当它找不到这个类或者初始化这个类（静态属性或代码块）失败时，就发生了java.lang.NoClassDefFoundError的错误。

更多有关此类的解释可以参考：

[http://blog.csdn.net/jamesjxin/article/details/46606307](http://blog.csdn.net/jamesjxin/article/details/46606307)

