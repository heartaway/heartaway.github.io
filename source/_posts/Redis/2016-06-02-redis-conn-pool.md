---
layout: post
title: Redis连接池的相关问题分析与总结
categories: Redis
description:
keywords: redis pool 连接池
date: 2016-06-02
---

>问题表象：服务端连接未释放

>问题背景：商品系统在运行过程中发生过一次Redis服务端连接数超限的问题。截图未保存，表现是：商品服务停掉，但RedisServer端看到的TCP连接任然存在，而且是 ESTABLISHED状态，导致的直接结果就是每次商品重启都会创建400个（minIdle=400）新的redis连接，而且停止的时候还不释放，重启几次之后RedisServer的连接就超过上限10000，无法再创建新的连接。

## 一、初步分析：
* TCP是可靠协议，断开是需要经过4次挥手，既然服务端的连接未释放，那十之八九是客户端没有发送断开请求；
* 即使客户端没有主动断开连接，但是这个连接实际上已经失效了，RedisServer为啥没有自动检测，而一直没有释放？

## 二、定位与排查过程：
### 问题一的分析： 
 首先怀疑的点是商品在停止的时候没有释放Jedis的连接，所以赶紧把JedisCluster的代码看了，果然有个close方法：


```java
@Override
public void close() throws IOException {
  if (connectionHandler != null) {
    for (JedisPool pool : connectionHandler.getNodes().values()) {
      try {
        if (pool != null) {
          pool.destroy();
        }
      } catch (Exception e) {
        // pass
      }
    }
  }
}
```

直接把这个方法注册到spring的destroy-method中：


​    
​    
​    
​    
​    
​    
​    
在性能测试环境上重新做下实现，发现此方法一直没有被调用，不解，看了下spring源码，调用destroy-method的触机有两个：

* 显示的调用org.springframework.context.support.AbstractApplicationContext.close()方法；
* 显示的调用org.springframework.context.support.AbstractApplicationContext.registerShutdownHook()，这样在程序正常停止的时候（kill，非kill -9 ）就可以触发shutdownHook的调用：


```java
public void registerShutdownHook() {
   if (this.shutdownHook == null) {
      // No shutdown hook registered yet.
      this.shutdownHook = new Thread() {
         @Override
         public void run() {
            doClose();
         }
      };
      Runtime.getRuntime().addShutdownHook(this.shutdownHook);
   }
}
```

check了我们的应用代码，这两个调用我们都没有做，所以如果需要触发Spring的destroy-method需要在想想办法。因为我们线上服务都是通过dubbo的Main启动，dubbo监管了Spring容器的生命周期管理，看看Dubbo的代码：(com.alibaba.dubbo.container.Main)


```java
if("true".equals(System.getProperty("dubbo.shutdown.hook"))) {
    Runtime.getRuntime().addShutdownHook(new Thread() {
        public void run() {
            Iterator i$ = var8.iterator();

            while(i$.hasNext()) {
                Container container = (Container)i$.next();

                try {
                    container.stop();
                    Main.logger.info("Dubbo " + container.getClass().getSimpleName() + " stopped!");
                } catch (Throwable var6) {
                    Main.logger.error(var6.getMessage(), var6);
                }

                Class t = Main.class;
                synchronized(Main.class) {
                    Main.running = false;
                    Main.class.notify();
                }
            }

        }
    });
}
```

这里发现一段重要代码，如果应用启动的时候指定了-Ddubbo.shutdown.hook=true，就会注册一个钩子函数，在Jvm进程正常停止的时候回触发container.stop()，而这些container中有一个就是com.alibaba.dubbo.container.spring.SpringContainer


```java
public class SpringContainer implements Container {
    private static final Logger logger = LoggerFactory.getLogger(SpringContainer.class);
    public static final String SPRING_CONFIG = "dubbo.spring.config";
    public static final String DEFAULT_SPRING_CONFIG = "classpath*:META-INF/spring/*.xml";
    static ClassPathXmlApplicationContext context;

    public SpringContainer() {
    }

    public static ClassPathXmlApplicationContext getContext() {
        return context;
    }

    public void start() {
        String configPath = ConfigUtils.getProperty("dubbo.spring.config");
        if(configPath == null || configPath.length() == 0) {
            configPath = "classpath*:META-INF/spring/*.xml";
        }

        context = new ClassPathXmlApplicationContext(configPath.split("[,\s]+"));
        context.start();
    }

    public void stop() {
        try {
            if(context != null) {
                context.stop();
                context.close();
                context = null;
            }
        } catch (Throwable var2) {
            logger.error(var2.getMessage(), var2);
        }

    }
}
```

springContainer的stop方法就是调用spring-context的close方法，最终会遍历每个bean的destroy-method。找打原因之后修改我们的启动脚本，加上参数：-Ddubbo.shutdown.hook=true，再次实验发现连接正常释放了，至此第一个问题的解决方案找到了，但是根本原因还没有找到，接下来通过抓包方式看能不能发现什么原因。

 

>Redis客户端断开连接，服务端连接未释放的根本原因分析：

这里选用了常用的抓包工具，tcpdump + wireshark（wireshark默认不支持redis协议，网上找了下，github上刚好有个同学共享了一个redis的协议解析插件：[http://blog.cheyo.net/257.html](http://blog.cheyo.net/257.html)，????）选用性能环境测试，在性能环境的RedisServer上抓包，同时监控RedisServier的连接数。


```shell
tcpdump -nnnvvv -s0 -X port 6379 and host 10.163.242.152 -i eth0 -w xxx.wap
 netstat -anp |grep ’10.163.242.152’ | wc -l

```

然后启动商品服务和停止商品服务，抓包观察。

我在做实现的时候搞个两个JedisCuster实例，每个实例的初始连接数（minIdle）都是1，有个实例注册了destroy-method，另一个实例未注册。

1.  第一个网络包是商品服务的启动过程，6个TCP包，报内容就是简单的TCP3次握手，创建了2个TCP链接

![](https://gw.alipayobjects.com/zos/skylark/e83ac639-7c60-469a-8afa-96bc5718fb9f/2018/png/e5579d08-d8f9-45bd-bea9-2fdc7c3998e0.png)

2.  第二个网络包是商品服务的停止过程，这里有看出区别了：第一个JedisCluster实例在结束的时候调用close命令，向RedisServe端发送了Quit指令，RedisServer返回OK。然后RedisServer主动发送FIN包，请求关闭连接，客户端ACK响应，然后客户端直接发送RST中断连接，这确保了服务端RedisServer可以断开连接。另一个JedisClsuter就直接给RedisServer发送一个RST，然后就没了。

![](https://gw.alipayobjects.com/zos/skylark/c371065a-7854-4479-b949-0de42a46ec54/2018/png/398cf6c3-e256-4025-9f3e-8a1a4a07385c.png)

 这里面有两个疑问？

(1). 为什么客户端没有遵循TCP的4次回收规范直接发送RST包中断连接？

(2).我们线上这么长时间都没有配置JedisCluster的close函数，启动脚本也没有配置-Ddubbo.shutdown.hook=true，为啥到现在也发现大问题，除了商品系统触发了？

先解答(2)的问题，我通过几次实现，发现即使我们什么都不做，直接停止服务，甚至kill -9 JVM线程，Client也会向RedisServer发送RST请求（我猜是OS自己做的，但具体没有再深入，有时间再看），但是并不是每次都发，会偶发性的丢包，一旦丢包，RedisServer就SB了，Client端私自把连接关闭了，然Server端却不知道，所以连接一直保持。

kill -9的网络包:

![](https://gw.alipayobjects.com/zos/skylark/6e148bf3-cf0e-448c-9d80-a5d7eaaae54b/2018/png/363b4efc-3011-4ff8-bf9f-5e81f316e7c3.png)

再看(1)的问题，为啥调用close的情况下Client还是给Server端返回RST包呢。查看了JedisCluser的源码，找到了原因：

redis.clients.jedis.Connection#connect

```java
socket = new Socket();
// ->@wjw_add
socket.setReuseAddress(true);
socket.setKeepAlive(true); // Will monitor the TCP connection is
// valid
socket.setTcpNoDelay(true); // Socket buffer Whetherclosed, to
// ensure timely delivery of data
socket.setSoLinger(true, 0); // Control calls close () method,
// the underlying socket is closed
// immediately
// <-@wjw_add
```

Jedis在创建Socket连接的时候配置了一个参数：socket.setSoLinger(true, 0); 这个参数的作用就是：在此socket调用close方法的时候不发送FIN包，而是直接发送RST包。

为什么Jedis要特意这么做呢？

简单来说就是TCP是一个可靠消息，而且TCP的4次挥手过程中存在一个半关闭状态，过程比较纠结：

>『假设Client端发起中断连接请求，也就是发送FIN报文。Server端接到FIN报文后，意思是说“我Client端没有数据要发给你了“，但是如果你还有数据没有发送完成，则不必急着关闭Socket，可以继续发送数据。所以你先发送ACK，“告诉Client端，你的请求我收到了，但是我还没准备好，请继续你等我的消息“。这个时候Client端就进入FIN_WAIT状态，继续等待Server端的FIN报文。当Server端确定数据已发送完成，则向Client端发送FIN报文，“告诉Client端，好了，我这边数据发完了，准备好关闭连接了“。Client端收到FIN报文后，“就知道可以关闭连接了，但是他还是不相信网络，怕Server端不知道要关闭，所以发送ACK后进入TIME_WAIT状态，如果Server端没有收到ACK则可以重传。“，Server端收到ACK后，“就知道可以断开连接了“。Client端等待了2MSL后依然没有收到回复，则证明Server端已正常关闭，那好，我Client端也可以关闭连接了。Ok，TCP连接就这样关闭了！』

TCP建立连接与释放链接的流程：

![](https://gw.alipayobjects.com/zos/skylark/871df9be-0d09-4449-a858-b302159aeff2/2018/png/d4bf9da8-6201-40a4-82a8-34bf4c97c39d.png)

**总结：**
​     
     一句话，就是TCP的正常关闭比较磨叽，而且会导致大量存在TIME-WAIT状态的连接。相对来说RST包就比较粗暴，直接告诉你『我要关闭连接了，老子不给你发数据，也不接收你的数据，你看着办吧』，然后自己就GG了。所以看Jedis的注释：

```java
Control calls close () method,
// the underlying socket is closed
// immediately
```

 关于RST的这个问题大家可以参考一下两个文章：

[http://xiaoz5919.iteye.com/blog/1685138](http://xiaoz5919.iteye.com/blog/1685138)（看评论）

http://docs.oracle.com/javase/1.5.0/docs/guide/net/articles/connection](http://docs.oracle.com/javase/1.5.0/docs/guide/net/articles/connection)（解释的很好，建议深读）

     至此第一个问题『因为TCP是可靠协议，断开是需要经过4次挥手，既然服务端的连接未释放，那十之八九是客户端没有发送断开请求』的根本原因基本调查清楚了，解决方案是在JVM加上启动参数：Ddubbo.shutdown.hook=true

### 问题二的分析：

接下来再看第二个问题『即使客户端没有主动断开连接，但是这个连接实际上已经失效了，RedisServer为啥没有自动检测，而一直没有释放？』，而且第二个问题还引发了上周我们线上的另一个问题。

  在商品系统出现连接未释放的问题之后，我紧急求救了啊泉，啊泉凭靠自身多年的运维经验很快反应这是RedisServe的一个配置的问题：


```java
# Close the connection after a client is idle for N seconds (0 to disable)
timeout 0
```

这个配置的意思是：RedisServer会自动关闭超过N seconds时间的idel连接，如果配置0那就是用不关闭。（我们现在所有的集群都是默认0）。啊泉当机立断，改成600。过了一会，Server端连接全部释放，应用恢复正常。

本来故事当这里就应圆满结束了，然后面几周却不消停，最初从财务系统开始，到订单系统、积分系统都开始偶发性报Jedis错误：   

![](https://gw.alipayobjects.com/zos/skylark/5d473e6b-6c35-4da3-9b34-16e253bea9b4/2018/png/94eb98c1-e1b2-426d-9a4a-ee38ce3aa64f.png)

还有商品系统一天到晚都偶发性报错：（部分原因可能是mget的原因)



```java
redis.clients.jedis.exceptions.JedisClusterMaxRedirectionsException: Too many Cluster redirections?
```

我们Check了下财务、订单、积分公用的Redis集群，除了内存使用量增长之外（单机超过1.3G），其他并无任何异常，初步猜想『这个Redis集群撑不住了』？拆，第二天就把购物车的Redis-DB拆出来了，但是错误依旧。难道还是内存的问题，当晚又把新的Redis-DB的key清理了下，把不用的key全删了，把使用内存降到350M左右。

公共Redis-DB的内存：

![](https://gw.alipayobjects.com/zos/skylark/9bad20ef-a810-4977-96ab-ec29b47791cf/2018/png/87ba7148-e01a-4fe0-bd05-95e17cbab107.png)


购物车的Redis-DB的内存：

![](https://gw.alipayobjects.com/zos/skylark/0de79097-cb5a-4be1-a347-cc768552f1e8/2018/png/be1e459a-944e-4044-bb7a-bec234f052bc.png)

 然降了内存之后错误依旧，SB了。中间翔把jedis的连接超时和请求超时时间都配置成20s，ms也解决了问题。但是我们讨论下来，这不是一个好的解决方案，这是饮鸩止渴的法子。我们需要从源头找问题，通过翻查资料和源码，初步判断是Client和Server的连接出了问题，感觉像是Jedis用了一个坏的连接去请求数据，然后Server端异常返回。Jedis官方有个贴了很详细的讨论了这个问题：


[https://github.com/xetorthio/jedis/issues/965](https://github.com/xetorthio/jedis/issues/965)。这篇文章很长，结论就是这个问题一般是RedisServer的连接坏了引发了的，坏了有几种可能：1. 网络抖动，连接失效。2. RedisServer这端主动关闭了，然Client端不知道（网络丢包）。所以解决方案有两个：

1.  RedisServer端配置timeout = 0，用不释放连接；

2.  Client需要定时检测和清理死链；

我们目前采用的解决方式是1，2都用了（2是最优解，因为网络环境相对复杂，网络断开的情况时有发生，所以客户端需要有检测死链的机制）。1的相关配置如下，主要有两个重要参数：


```java

private JedisPoolConfig getJedisPoolConfig(int redisMaxTotal, int redisMinIdle, int redisMaxWaitTime, int redisMaxIdle, boolean redisTestOnBorrow) {
    JedisPoolConfig poolConfig = new JedisPoolConfig();
    //最大连接数, 默认20个
    poolConfig.setMaxTotal(redisMaxTotal);
    //最小空闲连接数, 默认0
    poolConfig.setMinIdle(redisMinIdle);
    //获取连接时的最大等待毫秒数(如果设置为阻塞时BlockWhenExhausted), 如果超时就抛异常, 小于零:阻塞不确定的时间, 默认 - 1
    poolConfig.setMaxWaitMillis(redisMaxWaitTime);
    //最大空闲连接数, 默认20个
    poolConfig.setMaxIdle(redisMaxIdle);
    //在获取连接的时候检查有效性, 默认false
    poolConfig.setTestOnBorrow(redisTestOnBorrow);
    poolConfig.setTestOnReturn(Boolean.FALSE);
    //在空闲时检查有效性, 默认false
    poolConfig.setTestWhileIdle(Boolean.TRUE);
    //逐出连接的最小空闲时间 默认1800000毫秒(30分钟)
    poolConfig.setMinEvictableIdleTimeMillis(1800000);
    //每次逐出检查时 逐出的最大数目 如果为负数就是: idleObjects.size / abs(n), 默认3
    poolConfig.setNumTestsPerEvictionRun(3);
    //对象空闲多久后逐出, 当空闲时间 > 该值 且 空闲连接>最大空闲数 时直接逐出, 不再根据MinEvictableIdleTimeMillis判断 (默认逐出策略)
    poolConfig.setSoftMinEvictableIdleTimeMillis(1800000);
    //逐出扫描的时间间隔(毫秒) 如果为负数, 则不运行逐出线程, 默认 - 1
    poolConfig.setTimeBetweenEvictionRunsMillis(60000);
    return poolConfig;
}
```


配置这两个参数之后问题解决，且商品也不报错了，至此这两个问题算是解决了。

## 问题总结：

客户端与服务端之间的连接问题，可能存在客户端断开了连接服务端不知晓、服务端断开了连接客户端不知晓，为了解决这两个问题，需要做的就是服务端和客户端定期检查，客户端通过setTestWhileIdle(Boolean.True)、setTimeBetweenEvictionRunsMillis(xxx) 来定期检查方式死链；服务端通过设置超时时间来做到检查连接的问题。

后续：昨晚看VIP的运维分享，他们不建议RedisServer端的timeout配置成0，同样一个原因，网络异常的情况不可避免（或者应用端Kill -9），所以Server端同样需要有检查和清理死链的能力。



