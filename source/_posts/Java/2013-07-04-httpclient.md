---
layout: post
title: HTTP学习笔记（二）之HttpClient
categories: Java
description:
keywords: httpclient
date: 2013-07-04
---

#### HttpClient 是什么：

HttpClient 是 Apache Jakarta Common 下的子项目，用来提供高效的、最新的、功能丰富的支持 HTTP 协议的客户端编程工具包，并且它支持 HTTP 协议最新的版本和建议。

#### HttpClient 主要功能：

实现了所有 HTTP 的方法（GET,POST,PUT,HEAD 等）
支持自动转向
支持 HTTPS 协议
支持代理服务器等
HttpClient包下面都有那些内容：

#### HttpClient 的基本使用方法：
##### Get方法：

使用 HttpClient 需要以下 6 个步骤：

1. 创建 HttpClient 的实例
2. 创建某种连接方法的实例，在这里是 GetMethod。在 GetMethod 的构造函数中传入待连接的地址
3. 调用第一步中创建好的实例的 execute 方法来执行第二步中创建好的 method 实例
4. 读 response
5. 释放连接。无论执行方法是否成功，都必须释放连接
6. 对得到后的内容进行处理

Demo 示例：


```java
 //构造HttpClient的实例
  HttpClient httpClient = new HttpClient();
  //创建GET方法的实例
  GetMethod getMethod = new GetMethod("http://www.ibm.com");
  //使用系统提供的默认的恢复策略
  getMethod.getParams().setParameter(HttpMethodParams.RETRY_HANDLER,
    new DefaultHttpMethodRetryHandler());
  try {
   //执行getMethod
   int statusCode = httpClient.executeMethod(getMethod);
   if (statusCode != HttpStatus.SC_OK) {
    System.err.println("Method failed: "
      + getMethod.getStatusLine());
   }
   //读取内容
   byte[] responseBody = getMethod.getResponseBody();
   //处理内容
   System.out.println(new String(responseBody));
  } catch (HttpException e) {
   //发生致命的异常，可能是协议不对或者返回的内容有问题
   System.out.println("Please check your provided http address!");
   e.printStackTrace();
  } catch (IOException e) {
   //发生网络异常
   e.printStackTrace();
  } finally {
   //释放连接
   getMethod.releaseConnection();
  }
 }
```

注意点：

1. HttpClient的恢复策略是指在遇到异常时自动重试的机制，并且可以自定义（通过实现接口HttpMethodRetryHandler来实现）。通过httpClient的方法setParameter设置你实现的恢复策略，demo中使用的是系统提供的默认恢复策略，该策略在碰到IOException的时候将自动重试3次。
2. 用GetMethod将会自动处理重定向，如果想要把自动处理重定向去掉的话，可以调用方法setFollowRedirects(false)。
3. executeMethod返回值是一个整数，表示了执行该方法后服务器返回的状态码；
4. 获取响应内容的方式有三种：getResponseBody(返回二进制byte流)、getResponseBodyAsString(返回String)、getResponseBodyAsStream(这个方法对返回大量数据处理效率很高)；
5. 乱码：在使用getResponseBodyAsString容易出现乱码，影响响应内容字符编码的地方有两个：第一，Response的header设置，这里的设置编码可能与html内容编码不符，通过method对象的getResponseCharSet()方法就可以得到http头中的编码信息；第二，Respone的body内容中设置了`<meta http-equiv=”Content-Type” content=”text/html; charset=GBK”/>或<?xml version=”1.0″ encoding=”GBK”?>`标签，这样也可能与header编码冲突造成乱码。
6. 释放连接。无论执行方法是否成功，都必须释放连接。


```java
method.releaseConnection();
```

##### Post方法：
PostMethod基本与GetMethod类似；

不同点：

* POST和PUT，不支持自动重定向，因此需要自己对页面转向做处理。

Proxy代理服务的使用(host绑定)：
在日常工作中很多url的访问都是需要绑定host地址的，使用httpClient如何进行绑定访问？


```java
 HttpClient httpclient = new HttpClient();
            // 设置HTTP代理IP和端口
            httpclient.getHostConfiguration().setProxy(hostIp, hostIpPort);
```
支持Https：
HttpClient提供了对SSL的支持，允许HttpClient来打开Https连接。

有两种方法可以打开https连接，第一种就是得到服务器颁发的证书，然后导入到本地的keystore中；另外一种办法就是通过扩展HttpClient的类来实现自动接受证书。

方法1参考：[处理HTTPS协议](http://www.ibm.com/developerworks/cn/opensource/os-httpclient/#N10060)

方法2：

1. 实现自己的socket factory，必须实现SecureProtocolSocketFactory接口。
2. 自定义私有类 TrustAnyTrustManager implements X509TrustManager。
3. 定义Protocol ，将socket factory注册到 Protocol中。


```java
Protocol myhttps = new Protocol("https", new MySecureProtocolSocketFactory(), 443);
        Protocol.registerProtocol("https", myhttps);
```


Cookie处理：

通过state获取cookie：httpclient.getState().getCookies();


```java
    CookieSpec cookiespec = CookiePolicy.getDefaultSpec();
            cookies = cookiespec.match("login.daily.taobao.net", hostIpPort, "/", false,
                    httpClient.getState().getCookies());
```

CookieSpec接口代表了cookie管理的规范。cookiespec.match获取匹配路径下的cookie数组。

通过state设置cookie:


```java
httpclient.getParams().setCookiePolicy(CookiePolicy.RFC_2109);//RFC_2109是支持较普遍的一个，还有其他cookie协议
HttpState initialState = new HttpState();
Cookie cookie=new Cookie();
cookie.setDomain("www.balblabla.com");
cookie.setPath("/");
cookie.setName("多情环");
cookie.setValue("多情即无情");
initialState.addCookie(cookie);
httpclient.setState(initialState);
```

cookie协议：RFC 2109（过时的严格策略）、RFC 2965（严格策略的标准符合）、best-match：最佳匹配meta（元）策略(推荐使用)。

携带cookie进行页面访问：


```java
 postMethod.setRequestHeader("Cookie", cookieToString(cookies));
```
httpClient本身就可以保持cookie，可以通过同一个httpclient进行页面登录访问，但是对于有页面跳转的只能使用自带cookie进行模拟登录后访问。

通过HTTP上传文件：
httpclient使用了单独的一个HttpMethod子类来处理文件的上传，这个类就是MultipartPostMethod，该类已经封装了文件上传的细节，我们要做的仅仅是告诉它我们要上传文件的全路径即可。


```java
      ArrayList<Part> partArray = new ArrayList<Part>();
            for (int i = 0; i < params.size(); i++) {
                if (params.get(i) instanceof FileNameValuePair) {
                    partArray.add(new FilePart(params.get(i).getName(), ((FileNameValuePair) params.get(i)).getFile()));
                } else {
                    postMethod.addParameter(params.get(i).getName(), params.get(i).getValue());
                }
            }
            Part[] parts = new Part[partArray.size()];
            for (int i = 0; i < partArray.size(); i++) {
                parts[i] = partArray.get(i);
            }
            postMethod.setRequestEntity(new MultipartRequestEntity(parts, postMethod.getParams()));

            int statusCode = httpclient.executeMethod(postMethod);
```

通过HTTP模拟登录：


```java
public static String LoginDailyToGetPageWithHost(String nick, String psw, String scopeString, String url, List<NameValuePair> params, String hostIp, Integer hostIpPort) throws IOException {
        Protocol myhttps = new Protocol("https", new MySecureProtocolSocketFactory(), 443);
        Protocol.registerProtocol("https", myhttps);
        HttpClient httpClient = new HttpClient();
        Cookie[] cookies;
        LOGIN_URL = LOGIN_URL + "?redirectURL=" + url;
        PostMethod postMethod = new PostMethod(LOGIN_URL);
        postMethod.getParams().setParameter(HttpMethodParams.HTTP_CONTENT_CHARSET, "GBK");
        List<org.apache.commons.httpclient.NameValuePair> nameValues = new ArrayList<org.apache.commons.httpclient.NameValuePair>();
        nameValues.add(new org.apache.commons.httpclient.NameValuePair("TPL_username", nick));
        nameValues.add(new org.apache.commons.httpclient.NameValuePair("TPL_password", psw));
        nameValues.add(new org.apache.commons.httpclient.NameValuePair("action", "Authenticator"));
        /** httpclient post不支持方式重定向,fuck */
//        nameValues.add(new org.apache.commons.httpclient.NameValuePair("TPL_redirect_url", url));
        nameValues.add(new org.apache.commons.httpclient.NameValuePair("from", "tb"));
        nameValues.add(new org.apache.commons.httpclient.NameValuePair("event_submit_do_login", "anything"));
        nameValues.add(new org.apache.commons.httpclient.NameValuePair("TPL_checkcode", "8888"));
        nameValues.add(new org.apache.commons.httpclient.NameValuePair("loginType", "3"));

        postMethod.setRequestBody(nameValues.toArray(new org.apache.commons.httpclient.NameValuePair[nameValues.size()]));
        try {
            int status = httpClient.executeMethod(postMethod);
            System.out.println(status);
            CookieSpec cookiespec = CookiePolicy.getDefaultSpec();
            cookies = cookiespec.match("login.daily.taobao.net", hostIpPort, "/", false,
                    httpClient.getState().getCookies());
        } catch (Exception e) {
            System.out.println("Exception: " + e.toString());
            return null;
        } finally {
            postMethod.releaseConnection();
        }

        int postresult = -1;
        if (scopeString != null) {
            url = url + "&" + scopeString;
        }
        try {
            URI uri = new URI(url);
            postMethod.setURI(uri);
            // 如果不使用cookie 返回状态码 302，跳到i.daily.taobao.net去了，不能直接跳转到我们自己的页面
            postMethod.setRequestHeader("Cookie", cookieToString(cookies));
            // 设置HTTP代理IP和端口
            httpClient.getHostConfiguration().setProxy(hostIp, hostIpPort);
            if (params != null) {
                for (int i = 0; i < params.size(); i++) {
                    postMethod.addParameter(params.get(i).getName(), params.get(i).getValue());
                }
            }
            postresult = httpClient.executeMethod(postMethod);
        } catch (Exception e) {
            System.out.println("Exception: " + e.toString());
        } finally {
            postMethod.releaseConnection();
        }

        return postresult + postMethod.getResponseBodyAsString();
    }

    public static String cookieToString(Cookie[] cookie){
        StringBuffer cookieBuf = new StringBuffer(256);
        for(Cookie tmpCookie:cookie){
            cookieBuf.append(tmpCookie).append(";");
        }
        return cookieBuf.toString();
    }
```

MySecureProtocolSocketFactory 的实现：


```java
public class MySecureProtocolSocketFactory implements SecureProtocolSocketFactory {

    private SSLContext sslcontext = null;
    private SSLContext createSSLContext() {
        SSLContext sslcontext=null;
        try {
            sslcontext = SSLContext.getInstance("SSL");
            sslcontext.init(null, new TrustManager[]{new TrustAnyTrustManager()}, new java.security.SecureRandom());
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (KeyManagementException e) {
            e.printStackTrace();
        }
        return sslcontext;
    }
    private SSLContext getSSLContext() {
        if (this.sslcontext == null) {
            this.sslcontext = createSSLContext();
        }
        return this.sslcontext;
    }
    public Socket createSocket(Socket socket, String host, int port, boolean autoClose)
            throws IOException, UnknownHostException {
        return getSSLContext().getSocketFactory().createSocket(
                socket,
                host,
                port,
                autoClose);
    }
    public Socket createSocket(String host, int port) throws IOException,
            UnknownHostException {
        return getSSLContext().getSocketFactory().createSocket(
                host,
                port);
    }

    public Socket createSocket(String host, int port, InetAddress clientHost, int clientPort)
            throws IOException, UnknownHostException {
    	return getSSLContext().getSocketFactory().createSocket(host, port, clientHost, clientPort);
    }
    public Socket createSocket(String host, int port, InetAddress localAddress,
            int localPort, HttpConnectionParams params) throws IOException,
            UnknownHostException, ConnectTimeoutException {
        if (params == null) {
            throw new IllegalArgumentException("Parameters may not be null");
        }
        int timeout = params.getConnectionTimeout();
        SocketFactory socketfactory = getSSLContext().getSocketFactory();
        if (timeout == 0) {
            return socketfactory.createSocket(host, port, localAddress, localPort);
        } else {
            Socket socket = socketfactory.createSocket();
            SocketAddress localaddr = new InetSocketAddress(localAddress, localPort);
            SocketAddress remoteaddr = new InetSocketAddress(host, port);
            socket.bind(localaddr);
            socket.connect(remoteaddr, timeout);
            return socket;
        }
    }
    //自定义私有类
    class TrustAnyTrustManager implements X509TrustManager {
        public void checkClientTrusted(X509Certificate[] chain, String authType) throws CertificateException {
        }
        public void checkServerTrusted(X509Certificate[] chain, String authType) throws CertificateException {
        }
        public X509Certificate[] getAcceptedIssuers() {
            return new X509Certificate[]{};
        }
    }
}
```

HttpClient 使用中的坑：

1. 为什么 返回状态吗 为 200，但是 responebody = null ？
原因：在执行完 executeMethod 方法后，立即使用 getMethod.getResponseBodyAsString() 得到的是null，
通过在 executeMethod 后面打印出 返回体后在获取 返回内容，这时就能获取到了，添加 if (getresult == HttpStatus.SC_OK) 判断条件后也能获取到，说明 返回值内容不是立即写入的，有延迟。
解决方案： 添加 是否成功的判断。

2. 在进行请求的发生时，经常会遇到构建url参数时没有添加协议头:http://,这样会抛状态异常。
3. new HttpGet(url) 时出现 java.net.URISyntaxException 错误？
原因：url 中涉及了特殊字符，如‘｜’‘&’等。所以不能直接用String代替URI来访问。必须采用%0xXX方式来替代特殊字符。但这种办法不直观。所以只能先把String转成URL，再能过URL生成URI的方法来解决问题.

解决方案①：


```java
URL url = new URL(strUrl);
URI  uri = new URI(url.getProtocol(),null, url.getHost(),url.getPort(), url.getPath(), url.getQuery(), null);
HttpClient client    = new DefaultHttpClient();
HttpGet httpget = new HttpGet(uri);
```
有时候会越到 把http://头也给替换掉了，这样就会引发其它异常，所以还有一种比较保险的做法是使用queryString构建参数部分:


```java
 String url = "http://**.**.com/user/deployPackInfoDetail.htm";
        String queryString = "buildPackageId="+ buildPackageId;
        GetMethod getMethod = new GetMethod(url);
        getMethod.setQueryString(queryString);
```


