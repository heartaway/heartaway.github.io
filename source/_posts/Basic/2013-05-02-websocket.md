---
layout: post
title: Websocket 探究
categories: Basic
description:
keywords: websocket
date: 2013-05-02
---
**WebSocket：** 基于 HTML5 的一种浏览器与服务器之间的即时通讯解决方案(基于 TCP 连接的双向通道)；

**Java容器支持：**目前只支持 jetty 和 tomcat。

**应用场景：**即时通讯（网页游戏[双向异步消息模式]，网页聊天，微博等）

## 一、概念区分：
### HTTP 协议 与 TCP 协议区别：
HTTP协议是应用层协议，是用于www浏览的一个协，应用层协议包括HTTP协议，TELNET协议，WebSocket协议等。Http 协议是请求/响应范式的, 每一个 http 响应都是由一个对应的 http 请求产生的; http 协议是无状态的, 多个 http 请求之间是没有关系的.
TCP协议是传输层协议,是机器之间建立连接用的到的一个协议,传输层协议包括TCP协议，UDP协议。

### 长轮询 、短轮询 、长连接区别:
长轮询：首先长轮询并不等于长连接，长轮询还是采用的http短连接，只是服务器端把请求hold。
长连接：是多个 http 请求共用一个 tcp 连接; 这样可以减少多次临近 http 请求导致 tcp 建立关闭所产生的时间消耗。
短轮询：服务器收到请求不管是否有数据都直接响应 http 请求; 浏览器受到 http 响应隔一段时间在发送同样的 http 请求查询是否有数据;

### Http协议 与 webSocket协议关系：
WebSocket和Http协议一样都属于应用层的协议，webSocket在第一步进行握手时采用的是Http请求，第二步进行数据流传输是在TCP协议上操作，所以webSocket长连接 跟 Http长连接之间基本没有关系。本质上来说，WebSocket是不限于HTTP协议的，但是由于现存大量的HTTP基础设施，代理，过滤，身份认证等等，WebSocket借用HTTP和HTTPS的端口。由于使用HTTP的端口，因此TCP连接建立后的握手消息是基于HTTP的，由服务器判断这是一个HTTP协议，还是WebSocket协议。 WebSocket连接除了建立和关闭时的握手，数据传输和HTTP没丁点关系了。

![](https://yanmingming.files.wordpress.com/2013/05/2a213e4d-2cd6-4d8a-8be2-3bf50d8d7b64.png)

## 二、之前即时通讯的折衷方案：

1. 短轮询(Polling)：定时轮询，一定时间间隔浏览器向服务器发送请求。
2. 长轮询(Long Polling)：浏览器向服务器发送请求，有数据更新时反馈数据，如果服务器端没有新的数据需要发送，这里与Polling方法不同的是，服务器不是立即发送回应给浏览器，而是把这个请求保持住，等待有新的数据到来时，再来响应这个请求；当然了，如果服务器的数据长期没有更新，一段时间后，这个Get请求就会超时，浏览器收到超时消息后，再立即发送一个新的Get请求给服务器。然后依次循环这个过程。
3. 流：流技术方案通常就是在客户端的页面使用一个隐藏的窗口向服务端发出一个长连接的请求。服务器端接到这个请求后作出回应并不断更新连接状态以保证客户端和服务器端的连接不过期。

### 折衷方案的缺点：

1. 两次HTTP请求模拟双向通道，增加了编程复杂度。
2. 增加了服务器压力。
3. 制约应用的扩展性。


## 三、WebSocket原理：

![](https://gw.alipayobjects.com/zos/skylark/17625c30-c987-4090-9926-36a5fc709990/2018/png/07c0e5f9-d550-4bb5-9586-86554bbcdcc2.png)

1. 浏览器与服务器建立握手：
![](https://gw.alipayobjects.com/zos/skylark/9183678b-5633-493a-9b9f-512631ba8a6a/2018/png/cc7b882b-3327-4d99-9dfb-3d385572aaff.png)
2. 浏览器与服务器之间进行双向数据传输;

### Java中使用方法：

* 依赖 jetty 服务包：


```java
         <dependency>
            <groupId>org.eclipse.jetty.aggregate</groupId>
            <artifactId>jetty-all-server</artifactId>
            <version>8.1.9.v20130131</version>
         </dependency>
```

* IdeWebSocketServlet 继承 WebSocketServlet 抽象类：必须实现抽象接口方法 doWebSocketConnect()创建websocket的入口.定义 WebSocket连接缓存-new ConcurrentHashMap<String, WebSocket>()；用来管理所有webSocket连接，每次新建连接都会写入map，断开连接也会从map中移除这个元素。
    *  建立WebSocket连接的过程：
![](https://yanmingming.files.wordpress.com/2013/05/c8affbd4-5659-41c6-aba7-6613d44715fc-1.png)

* 定义自己的WebSocket对象,实现 WebSocket.OnTextMessage 接口,比如 SessionWebSocket：实现onOpen、onClose、onMessage 方法。当事件类型较多时，定义一组事件列表， onMessage 中根据事件类型执行相应事件的具体方法。
* 客户端中Js的写法：
  
```java
function connect() {
            var target = document.getElementById('target').value;
            if (target == '') {
                alert('Please select server side connection implementation.');
                return;
            }
            if ('WebSocket' in window) {
                ws = new WebSocket(target);
            } else if ('MozWebSocket' in window) {
                ws = new MozWebSocket(target);
            } else {
                alert('WebSocket is not supported by this browser.');
                return;
            }
            ws.onopen = function () {
                setConnected(true);
                log('Info: WebSocket connection opened.');
            };
            ws.onmessage = function (event) {
                log('Received: ' + event.data);
            };
            ws.onclose = function () {
                setConnected(false);
                log('Info: WebSocket connection closed.');
            };
        }

        function updateTarget(target) {
            if (window.location.protocol == 'http:') {
                document.getElementById('target').value = 'ws://' + window.location.host + target;
            } else {
                document.getElementById('target').value = 'wss://' + window.location.host + target;
            }
        }
```

### WebSocket优势：
* 从服务器上主动给客户端推送信息；
* 比Http协议传输效率高；
### WebSocket劣势：
* 浏览器支持的兼容性不好，对于IE，支持IE10；

### WebSocket为什么要把message分成frame：
因为HTTP协议有chunk功能，可以让服务器一边生数据，一边发。而websocket协议也考虑到了这点。如果没有framing功能，那么我必须知道整个message的长度之后，才能开始发送message的data。

### 长连接误解：
关于 http 长连接一个误解就是服务器主动推送数据, 这个在 http 协议下是无法实现的, 因为 http 请求/响应范式决定的, http 中服务器返回数据必须要有一个浏览器端的请求对应, 服务器无法主动推送给浏览器数据.

### 长短轮询策略：
不管 http 长轮询还是 http 短轮询 保证同一个用户在多 tab 下只存在一个定时查询是有好处的, 这可以通过在浏览器端缓存数据解决, 在 http 响应后在浏览器端缓存数据, 并设置一个有效期, 然后在每次发送 http 请求时检查是否有有效数据, 没有则发送请求获取。


