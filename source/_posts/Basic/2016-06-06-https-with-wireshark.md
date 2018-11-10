---
layout: post
title: Wireshark监控分析Https的交互过程
categories: Basic
description:
keywords: https Wireshark
date: 2016-06-06
---

首先介绍一下Https在Client与Server之间的理论交互过程：
![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/fbcfb4b7d254a68b4e39ddbeef776927.png)

试想，我们期望在网络中传输的数据是被加密的，并且这个数据包在客户端和服务器端都能进行加密和解密，那么需要对称加密算法，客户端和服务端都需要有一个加解密Key（对称加密），对数据包进行加解密，那么这个Key如何产生并在在网络中安全的传递呢？假设这秘钥是有服务端产生，分配给客户端，为了保证安全，需要在派送Key的过程中进行加密，那么客户端在一无所知的情况下，是无法打开这个加密的数据包的，此方案不行。假设我们引入非对称加密算法，把公钥在网络上传输(传给客户端)，私钥只保留在服务端，让客户端生成一个加密Key，并把加密Key通过公钥进行加密，加密后的数据发送给服务端，服务端使用私钥解开数据，提取加密Key，这样服务端和客户端以后的交互都可以使用这个加密Key进行数据交互了。

这个过程可以参考：[http://www.jianshu.com/p/15703d8c34e9](http://www.jianshu.com/p/15703d8c34e9)

### 抓包监控分析：
首先安装抓包工具Wireshark(最新版为2.0.3)，Mac OS 10.11版本还需要安装X11，OS X 不再随附 X11，但 XQuartz 项目提供适用于 OS X 的 X11 服务器和客户端库，所以需要下载 XQuartz。
打开Wireshark > 捕获 > 选项，选择 本地网卡，开启监控数据捕获。
这里以腾讯云 [https://www.dnspod.cn](https://www.dnspod.cn) 为例（首次打开此网站，所以可以捕获到证书获取流程），由于本地有很多其他的应用都会进行网络请求，为排除干扰，查询了腾讯云（[https://www.dnspod.cn](https://www.dnspod.cn) ）的IP地址：59.37.116.101，然后在Wireshark 的过滤器中输入：


```java
ip.src==59.37.116.101  or ip.dst==59.37.116.101
```

表示只监控IP的起始地或者目的地为腾讯云的数据请求，如图所示：

![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/f3fea5e73ee24a7e97bf9f43a7e29534.png)

当然，这里还是可能会有其他tcp流的干扰，找到一个Client Hello 的tcp请求，查看Transmission Control Protool,找到Stream Index 序号，这个Stream Index 是Wireshark标识一次完整的TCP通讯。

![](/images/posts/20160606/wirdshark-tencent-cloud-tcp-stream-get.png)

所以，调整我们的过滤器为：

```java
ip.src==59.37.116.101  or ip.dst==59.37.116.101 and tcp.stream eq 28
```

#### 1. 三次握手建立TCP链接：
从请求流中，我们可以清晰的看到在建立TLS请求之前，进行了三次TCP请求，这就是我们常说的TCP建立连接的“三次握手”：

![](/images/posts/20160606/tcp-third-handshake.png)

##### 第一次握手：
客户端请求建立连接（Flags=SYN，Seq = 0 ）：

![](/images/posts/20160606/tcp-first-handshake.png)

SYN标识初始化连接，序列号这里为0，注意括号里面的备注，说0是相对序列号，并不是真正的实际序列号，相对序列号只是方便我们查看。
##### 第二次握手：
服务端发送确认建立连接数据包给客户端（Flags=SYN，ACK，seqNum=0,ackNum=1）：

![](/images/posts/20160606/tcp-second-handshake.png)

##### 第三次握手：
客户端再次发送确认包(ACK) SYN标志位为0,ACK标志位为1.并且把服务器发来ACK的序号字段+1,放在确定字段中发送给对方.并且在数据段放写ISN的+1

![](/images/posts/20160606/tcp-third-time-handshake.png)

当然还可以通过WireShark自带的统计图形工具查看链接交互过程：

![](/images/posts/20160606/https-tcp-graph.png)

有关TCP建立链接和释放连接的过程，可以参考：[TCP/IP 之 大明王朝邮差](http://mp.weixin.qq.com/s?__biz=MzAxOTc0NzExNg==&mid=2665513094&idx=1&sn=a2accfc41107ac08d74ec3317995955e&scene=23&srcid=0605GAETrvLCgoti5SQnXy0o#rd)

#### 2. Client Hello：

![](/images/posts/20160606/https-tcp-client-hello.png)

我们可以看到，从浏览器发出的第一个字节为0×16（十进制的22），它表示了这是一个“握手”记录。

整个握手记录被拆分为数条信息，其中第一条就是我们的客户端问候（Client Hello），即0×01。在客户端问候中，有几个需要着重注意的地方：

##### 随机数（Random）：


![](/images/posts/20160606/https-tcp-client-hello-random.png)


在客户端问候中，有四个字节以Unix时间格式记录了客户端的协调世界时间（UTC）。协调世界时间是从1970年1月1日开始到当前时刻所经历的秒数。在这个例子中，a1 60 cf bd 就是协调世界时间。在他后面有28字节的随机数，在后面的过程中我们会用到这个随机数。

##### SID（Session ID）：

![](/images/posts/20160606/https-tcp-client-hello-sid.png)

##### 数（Random）：


如果出于某种原因，对话中断，就需要重新握手。为了避免重新握手而造成的访问效率低下，这时候引入了session ID的概念， session ID的思想很简单，就是每一次对话都有一个编号（session ID）。如果对话中断，下次重连的时候，只要客户端给出这个编号，且服务器有这个编号的记录，双方就可以重新使用已有的”对话密钥”，而不必重新生成一把。

在这里，SID是一个空值（Null）。说明我们是第一次访问这个网站，如果我们在几秒钟之前就登陆过了www.baidu.com，我们有可能会恢复之前的会话，从而避免一个完整的握手过程。

session ID是目前所有浏览器都支持的方法，但是它的缺点在于session ID往往只保留在一台服务器上。所以，如果客户端的请求发到另一台服务器，就无法恢复对话。session ticket就是为了解决这个问题而诞生的，目前只有Firefox和Chrome浏览器支持。后面关于https优化部分会着重介绍session ticket。

##### 密文族（Cipher Suites）：

![](/images/posts/20160606/https-tcp-client-hello-cipher.png)

密文族是浏览器所支持的加密算法的清单。RFC2246中建议了很多中组合，一般写法是”密钥交换算法-对称加密算法-哈希算法，以“TLS_RSA_WITH_AES_256_CBC_SHA”为例，TLS为协议，RSA为密钥交换的算法，AES_256_CBC是对称加密算法（其中256是密钥长度，CBC是分组方式），SHA是哈希的算法。浏览器支持的加密算法一般会比较多，而服务端会根据自身的业务情况选择比较适合的加密组合发给客户端。（比如综合安全性以及速度、性能等因素）

##### Server_name扩展：

![](/images/posts/20160606/https-tcp-client-hello-servername.png)

当我们去访问一个站点时，一定是先通过DNS解析出站点对应的ip地址，通过ip地址来访问站点，由于很多时候一个ip地址是给很多的站点公用，因此如果没有server_name这个字段，server是无法给与客户端相应的数字证书的，Server_name扩展则允许服务器对浏览器的请求授予相对应的证书。

#### 3. Server Hello
![](/images/posts/20160606/https-tcp-server-hello.png)


服务器在收到client hello协议的握手请求后，会先回复一个ACK，标识client hello握手数据包收到了，然后向客户端发送Server hello协议数据包，这个协议包里面包含了Session ID,Cipher Suite,Random 等数据。

我们记得在client hello里面，客户端给出了21种加密族，而在我们所提供的21个加密族中，服务端挑选了“TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256”。这就意味着服务端会使用ECDHE-RSA算法进行密钥交换，通过AES_128_GCM对称加密算法来加密数据，利用SHA256哈希算法来确保数据完整性 。

服务端对于session ID一般会有三种选择：

1. 恢复的session ID：我们之前在client hello里面已经提到，如果client hello里面的session ID在服务端有缓存，服务端会尝试恢复这个session；
2. 新的session ID：这里又分两种情况，第一种是client hello里面的session ID是空值，此时服务端会给客户端一个新的session ID，第二种是client hello里面的session ID此服务器并没有找到对应的缓存，此时也会回一个新的session ID给客户端；
3. NULL：服务端不希望此session被恢复，因此session ID为空。

#### 4. Certificate
![](/images/posts/20160606/https-tcp-certificate.png)

我们知道为了安全的将公钥发给客户端，服务端会把公钥放入数字证书中并发给客户端（数字证书可以自签发，但是一般为了保证安全会有一个专门的CA机构签发），所以这个报文就是数字证书，3259 byte 就是证书的长度。

其次，服务端还向客户端发送了协议为Server Hello Done的报文，意思是告诉客户端hello结束，并且不要求验证客户端的证书。（我们知道TLS协议中验证证书可以是双向的，即服务端也要验证客户端的身份来防止客户端的伪冒，但是这种场景在一般基于web的https中很少(通过U盾的网银是例外，U盾其实就是客户端证书，但这样也非常繁琐)，因为基于web的应用客户数量大，很难为每个客户去提供相应的数字证书。但是对于一些企业之间的对接，出于安全考虑，很多情况下会采用双向认证的方式，因为对于两个企业来说也不存在client、server端的说法。）

#### 5. 客户端验证证书
当客户端收到服务器发来的证书后会进行校验来确保证书是真实的服务器发来的，那么如何验证其真实性呢？主要靠的就是数字签名。algorithmIdentifier（签名的加密算法）为：SHA_RSA（注意这里的加密算法是证书中自带的，并非我们之间client、server里面的Cipher Suites中的算法，Cipher Suites中的算法只有密钥交换算法、对称数据加密算法、校验完整性的哈希算法，不包括相关签名的算法），而下方的encrypted部分就是数字签名，是由CA的私钥加密的，数字证书制作这块前面一篇文章已经介绍过了，这里就不多说了。客户端收到数字证书中的数字签名后，由于证书的签名都是由上一级来完成的，因此会利用上一级证书提供的公钥进行解密，解密后得到签名值和自己再次哈希证书主题的值进行比对，如果两个值是一致的话，则认证通过。

#### 6. Client Key Exchange
客户端验证通过证书后，客户端将采用服务器给出的加密算法以及公钥将后面用于加密数据的对称密钥进行加密并发送给服务器。其实密钥交换这步远没有想象中的那么简单，主要有以下几个步骤（以下为RSA算法密钥交换的过程，腾讯云用的密钥交换算法为ECDHE-RSA，比RSA更为复杂，这里就先介绍RSA算法）：


1. 客户端生成premaster_secret，首先对称密钥为了保证安全一定是随机密钥，一般的系统或者浏览器都会构建它自己的伪随机数发生器，之所以称之为伪随机数是因为真正意义上的随机数算法并不存在，这些函数还是利用大量的时变、量变参数来通过复杂的运算生成相对意义上的随机数，但是这些数之间还是存在统计学规律的，只是想要找到生成随机数的过程并不那么容易。在进行密钥交换的时候，客户端会利用server hello里面带的随机数2（client hello里面的随机数我们称为随机数1，这是为了方面后面master_secret）生成premaster_secret，premaster_secret长度为48个字节，前2个字节是协议版本号，剩下的46个字节填充随机数。


2. 注意，对称加密不会直接用这个premaster_secret进行加密。因为这个premaster_secret完全由客户端来提供，完成没有服务端的相关信息的参与，因此客户端会利用premaster_secret生成master_secret，然后再用master_secret生成对称密钥算法、MAC 算法的密钥。
   
   >master_secret =PRF(pre_master_secret, “master secret”, ClientHello.random +ServerHello.random)
   
   pre_master_secret就是我们之前传送的随机密码串，”mastersecret”是一串ASCII码，再加上之前提到的random1和random2。PRF是在规范中约定的伪随机函数，它将密钥、ASCII码标签、哈希值整合在一起。各有一半的参数分别使用MD5和SHA-1获取哈希值。这是一种十分明智的做法，即使是想要单单破解相对简单MD5和SHA-1也不是那么容易的事情。而且这个函数会将返回值传给自身直至迭代到我们需要的位数。关于PRF的具体细节问题


3. 客户端得到master_secret后，根据协议约定，我们需要利用PRF生成这个会话中所需要的各种密钥，称之为“密钥块”（key block），密钥块用于构成以下密钥：
client_write_MAC_secret、server_write_MAC_secret （MAC 算法的密钥）
client_write_key、server_write_key （对称算法密钥or会话密钥）
client_write_IV、server _write_IV （初始化向量，运用于分组对称加密，如果mode是CBC则需要这个值，这个后面降到算法那篇会重点介绍）
以上的6个密钥都是通过PRF运算出来的，具体的运算方法比较复杂，后面讲到算法那张会单独介绍。


4. 至此，经过前几步客户端已经完成了相关密钥的生成。此时客户端通过证书中的服务端的公钥将premaster_secret加密后发给服务端，如下图报文中的Client Key Exchange，通过RSA非对称密钥算法利用数字证书中获得的公钥将premaster加密发给服务端。（这里是一个RSA作为密钥交换算法的报文，需要注意腾讯云我们从server-hello中就知道了腾讯云用了更为复杂的ECDHE_RSA算法用于密钥交换算法，所以我们在访问腾讯云的同样的报文中看到的EC Diffie-Hellman Client Params的算法。）

    ![](/images/posts/20160606/https-tcp-client-key-exchanger.png)

5. 在发送Client Key Exchange报文的同时，客户端还会回一个Change Cipher Spec的报文（如上图），意思就是通知服务端后续报文将采用协商好的密钥和加密套件进行加密和 MAC计算。


6. 最后，客户端计算已交互的握手消息（除 Change Cipher Spec 消息外所有已交互的消息）的 Hash值，通过前面生成的密钥client_write_MAC_secret对hash值进行加密得到MAC，并通过HandShake Messag报文发送给服务端（如上图）。


7. 服务端此时会收到一个premaster_secret，以及一个客户端发来的MAC。利用Change Cipher Spec报文中协商好的密钥和算法，服务端也会计算出用于加密的数据的密钥以及MAC的加密密钥。此时服务端会用利用同样的方法计算已交互的握手消息的 Hash 值，并用client_write_MAC_secret解密客户端发来的MAC，两者进行比较，如果二者相同，且 MAC值验证成功，则证明服务端密钥交换协商成功。
    ![](/images/posts/20160606/https-tcp-secret-encrypted.png)


8. 当服务端密钥交换协商成功的同时，会同样回客户端一个Change Cipher Spec报文（如上图），通知客户端后续报文将采用协商好的密钥和加密套件进行加密和 MAC计算。（注意，这个报文前面还有一个New Session Ticket的报文，这个是百度对于https的优化之一，后面会详细讲，在最为基础的https连接中是没有这个报文的。）


9. 同样服务端也会计算已交互的握手消息的 Hash值，通过前面生成的密钥server_write_MAC_secret对hash值进行加密得到MAC，并通过HandShake Message报文（如上图）发送给客户端。


10. 由于客户端在发送消息给服务端前就已经计算出了数据加密的对此密钥，因此只需要验证服务端返回的MAC即可，所以此时客户端端会用利用同样的方法计算已交互的握手消息的 Hash 值，并用server_write_MAC_secret解密服务端发来的MAC，两者进行比较，如果二者相同，且 MAC值验证成功，则证明客户端密钥交换协商成功。


11. 至此，整个密钥交换的协商全部完成，开始传输应用层的数据。


#### 7. 数据传输
经过之前6大步骤，SSL的连接建立完成，开始有客户端发起Application Data，实际上下图中的encrypted application data就是http协议的数据经过client_write_key、server_write_key进行加密后的传输，我们通过一些工具解密后，发现应用层实际就是标准的http协议，也就是说SSL实际上就是在http协议外面套了层壳


![](/images/posts/20160606/https-tcp-data-transport.png)

请求链路分析文件（可直接通过wireshark打开）：
[http://pan.baidu.com/s/1hsi5U4W](http://pan.baidu.com/s/1hsi5U4W)


#### 参考链接：
[http://www.jianshu.com/p/a766bbf31417](http://www.jianshu.com/p/a766bbf31417)

[http://www.cnblogs.com/TankXiao/archive/2012/10/10/2711777.html](http://www.cnblogs.com/TankXiao/archive/2012/10/10/2711777.html)

