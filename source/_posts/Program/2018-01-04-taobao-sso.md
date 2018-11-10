---
layout: post
title: Taobao SSO 跨域登录过程解析
categories: Program
description:
keywords: SSO taobao 跨域 跨域登录
date: 2018-01-04
---
今年的双十一和双十二已经告一段落，你是否买到了你想要的宝贝呢？我们知道双十一是天猫的主场，双十二是淘宝的主场，你有没有注意到你在登录了淘宝后，访问天猫或者飞猪，你还是处于登录态的，但是我们知道cookie是不能跨域的，那么阿里是如何做到了多域名下的登录态同步呢？接下来我们通过抓包进行请求解析来了解这个过程。

## 基础知识：
1. 如果忘了Cookie和Session的区别，那么建议你先回顾一下，可以参考：https://github.com/astaxie/build-web-application-with-golang/blob/master/zh/06.1.md
2. 如果不知道为什么需要鉴权，为什么需要SSO，为什么需要跨域登录，建议你先阅读上一篇文章“系统权限控制”。


## 测试过程：
1. 访问www.taobao.com 请求登录，跳转到 login.taobao.com
    * 输入用户名和密码后，登录成功，302 回调到www.taobao.com页面
    * Post 表单到 login.taobao.com , response 为 set-cookie，并通过redirectURL 跳会www.taobao.com首页；
2. 访问 www.tmall.com
    * www.tmall.com页面响应中发起新的请求 tmcc.tmall.com/pass.com
3. 请求页面 https://tmcc.tmall.com/pass.htm
    * 响应为 302 跳转到： https://login.taobao.com/jump?target=https://tmcc.tmall.com/pass.htm?tbpm=1
    * tbpm=1表示：进行tbsession的跨域同步；
4. 请求页面 https://login.taobao.com/jump?target=https://tmcc.tmall.com/pass.htm?tbpm=1
    * 在请求login.taobao.com/jump时，会携带上taobao.com域下的cookie信息
    * 响应为302跳转到：https://pass.tmall.com/add?_tb_token_=eef03e35fbe&u......&target=https%3A%2F%2Ftmcc.tmall.com%2Fpass.htm%3Ftbpm%3D1
    * 服务器端把taobao域下的cookie信息拼接到了302跳转的url 的 query string上。
5. 请求页面 https://pass.tmall.com/add?_tb_token_=eef03e35fbe&u......&target=https%3A%2F%2Ftmcc.tmall.com%2Fpass.htm%3Ftbpm%3D1
    * 携带cookie 信息的query string 参数请求 tmall.com域名下的信息，请求添加cookie。
    * 响应中把请求中的cookie信息set 到 浏览器cookie中，以此完成tmall.com域名下的cookie同步；
    * 响应状态未302 重定向到 https://tmcc.tmall.com/pass.htm?tbpm=1
6. 请求页面https://tmcc.tmall.com/pass.htm?tbpm=1 302 跳转到 https://tmcc.tmall.com/pass.htm

抓包信息：
![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/87d35687c06bdf7d01ea78c3aa1f8c7f.png)
图中隐藏了非关键请求，比如页面的静态资源等；

同样：

1. 访问 www.alitrip.com，会同步请求https://ffa.alitrip.com/userInfo.htm的请求
2. 请求页面https://ffa.alitrip.com/userInfo.htm
    * 响应为 302 跳转到：https://login.taobao.com/jump?target=https%3A%2F%2Fffa.alitrip.com%2FuserInfo.htm
3. 请求页面https://login.taobao.com/jump?target=https%3A%2F%2Fffa.alitrip.com%2FuserIn
    * 在请求login.taobao.com/jump时，会携带上taobao.com域下的cookie信息
    * 响应为302跳转到：https://pass.alitrip.com/add?_tb_token_=eef03e35fbe&uss=UtBTusG4…
    * 服务器端把taobao域下的cookie信息拼接到了302跳转的url 的 query string上。
4. 请求页面https://pass.alitrip.com/add?_tb_token_=eef03e35fbe&uss=UtBTusG4
    * 携带cookie 信息的query string 参数请求 alitrip.com 域名下的信息，请求添加cookie
    * 响应中把请求中的cookie信息set 到 浏览器cookie中，以此完成tmall.com域名下的cookie同步;

示意图：
![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/ad93fbcaff26e6b4bc3ca3af620468e9.png)
1. 访问原始url： tmcc.tmall.com
2. 重定向，访问login.taobao.com/jump
3. 重定向，访问pass.tmall.com
4. 重定向，访问原始url（带有同步标识tbpm）
5. 重定向，访问原始url（去掉同步标识tbpm）

如果我们taobao.com域下没有登录cookie，通过在login.tmall.com页面进行登录，那么cookie的传递是怎么样的呢？
通过测试，发现在请求login.tmall.com的时候会同步请求login.taobao.com然后cookie依然是通过taobao域同步到tmall域名，也就是cookie的同步是单向；


当然，这个过程是正向流程，那退出登录的逆向流程是怎么样的呢？会同步请求login.taobao.com/clear, 通过set-cookie 清楚session cookie（会话cookie），然后进行 302 跳转到 https://pass.tmall.com/clear 进行cookie清理，然后 302 跳转到 https://pass.etao.com/clear 进行cookie清理，然后 302 跳转到https://pass.etao.com/clear 进行 cookie清理，如此重复知道所有域名cookie进行清理完毕。
![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/eafb7fc66c6139c50a6a8bbb5bd89e58.png)
通过测试发现淘系的系统所有的退出登录都是走login.taobao.com/member/logout.jhtml , 然后通过一些列302 跳转pass系统进行登录态的清理。 登录态在到阿里云、支付宝是不通的,因为阿里云和支付宝的账号体系不一样。


## 禁用Cookie：
可以看出整个跨域登录依赖的是cookie信息的传递与跨域设置，如果出于安全考虑，我们禁用了cookie，是否还能正常工作呢？经过测试，发现禁用cookie后，跨域自动登录不能正常执行了，跨域请求后对于受限的访问请求还是会自动跳登录。由于设计上没有考虑cookie禁用的情况，淘宝的登录页面竟然无法进入，一直循环跳转登录页面。

## 分布式Session 的常见解决方案：
1. 通过cookie进行共享；
2. 借助第三方进行存储，比如缓存；
3. 不容服务器之间进行session同步；

这里面涉及到阿里两个重要级产品：tbsession、passcookie。

1. tbsession：用来解决多应用间session共享存储与同步问题；tbsession采用的是 方法1 + 方法2的结合；
2. passcookie：用来解决不同域名之间cookie同步的问题，以及决定同步那些cookie；

## 思考：
问：为什么有了cookie 还需要 session？

答：cookie存储本身具备一些优势，比如信息存储在客户端，分散了资源消耗，cookie可以在客户端进行持久化存储（cookie在客户端分为：Permanent Cookies，Session Cookie）。    主要是只使用cookie作为资源访问的鉴权记录具备不安全性，容易引起CSRF（跨站请求伪造），比如攻击者劫持登录后的cookie信息进行页面操作，此时服务器以为还是用户自身在登录态下的本人操作。当然，只是简单的使用session也并不能彻底解决CSRF，使用session只是把用户的登录态信息保存在服务器端，客户端cookie往往会记录一个JSessionId 用来标识当前会话ID，jsessionid在网络中传输还是存在被劫持的可能性（参考下面的session劫持），所以需要配合响应的解决手段防止CSRF的发生。其次，cookie的使用在大小和条数上限制，大于需要存储大量用户态信息的场景下已经不够用了，此时需要借助session在服务端的存储设备来实现。
![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/877347f9c7fc1318a494640818519c2a.png)

问：登录后会会话中的每一次访问受限资源都需要访问验证？

答：是的，因为每一次请求都无法确认你的身份，所以为了降低复杂认证授权的过程，通过sessionId 来标识某一会话；如果我们的Cookie信息泄露，那么不法分子就可以使用我们包含登录态的Cookie进行访问我们的受限资源（比如拷贝登录后的cookie信息通过postman请求需要登录后才能查看的信息），即便是我们丢增删改的请求采用了crsf_token ,不法分子还是可以看到我们的信息，信息已经造成了泄露。所以对于持久化Cookie，尽量设置为httpOnly，不允许通过JS脚本读取Cookie信息。还有就是使用https协议代替http协议（从tcp到ssl），这样不法分子劫持了请求信息，也无法破解请求信息的内容。

问：是否采用了https就不需要防范CSRF了？

答：不是的，我们采用https，是在数据传输层对请求进行进行加密后发送，但是如果CSRF是在浏览器的本地读取我们的Cookie信息（存储在我们浏览器中本地的Cookie信息确实明文的），还是可以读取到我们明文的Cookie信息。采用https只是防止了请求在传输中拦截后窃取我们的Cookie进行攻击而已。


## Session 劫持与防范：
这篇文章详细阐述了服务器端不做任何处理下的sessionId劫持的案例。
https://github.com/astaxie/build-web-application-with-golang/blob/master/zh/06.4.md

session 劫持的防范措施：

1. 使用POST 请求 + Referer验证；
    * 此方式简单容易实现；
    * 弊端：1.所有请求都是用POST方式不符合规范，比如RestFul。2.很多场景下请求是无法携带referer的；
2. 使用csrf_token：在涉及增删改的请求中，带上一个和服务器端同步的随机数(token)，然后在服务器端做校验。由于这个token是变化的，同时具有私密性，只会内嵌到当前用户的页面中，因此可以起到防止CSRF的作用，比如淘宝域下cookie中的_tb_token_参数。

csrftoken的整个生命周期一般是这样的：

* 后端针对每一个用户生成唯一的csrftoken后    
* 将csrftoken存储在服务器端，且和当前用户一一绑定
* 当用户访问页面时，将csrftoken埋入前端页面
* 用户提交请求时，需要在请求中附带上csrftoken
* 后端对接收到的csrftoken进行校验，判断请求是否合法
* 根据请求，后端判断是非需要更新csrftoken

使用csrf_token 方案目前来说是最有效的安全措施，但是方案实施起来也相对复杂很多，需要考虑csrf_token的存储、更新、埋点以及校验等逻辑。

## 本质思考：
SSO 跨域统一登录的本质就是【登录态信息的跨域共享，为了实现共享采用了复制的方式】，这跟我们现实世界中为了达到知识共享，需要把知识转化为书籍，然后通过书籍印刷进行分发。

## 抓包工具介绍：
1. 工具采用 mac 下的 Charles；
2. https由于安全限制无法抓取到具体的内容，所以需要通过一些列的设置才能抓到请求数据；
3. Charles -> Help -> SSL Proxying -> Install Charles Root Certificate
4. Proxy -> SSL Proxying Setting -> SSL Proxying 添加正则域名

