---
layout: post
title: Capture Html Content After Render
categories: Java
description:
keywords: phantomJS
date: 2017-09-21
---
在前端技术越来越注重用户体验的潮流下，异步数据渲染组件成为主流，但是异步渲染给爬虫和相关需要使用页面异步渲染后Dom结构的需求来说，变得很麻烦。要想获取渲染后的页面内容，常见的方法有：

1. 使用类似于PhantomJS的Headless浏览器，模拟页面渲染，获取页面HTML；
2. 分析Ajax请求，获取请求后的数据。

对于千变万化的页面结构来说，第一种方法更具备通用性，首选需要安装PhantomJS，然后编写js脚本，最后就是运行脚本获取结果啦。

比如我编写了一个loadPage.js,用于支持页面截图或者页面渲染后的HTML结构获取：

```java
var args = require('system').args;
var page = require('webpage').create();
var fs = require('fs');
var url;
var fileName = "";
var cookieJson = "";
var type = "html";
var viewWidth = 1024;
var viewHeight = 768;


args.forEach(function(arg, i) {
    if(i === 1){
      url = arg;
    }

    if(i === 2){
      type = arg;
    }

     if(i === 3){
      fileName = arg;
    }

    if(i === 4){
      cookieJson = arg;
    }

    if(i === 5){
      viewWidth = arg;
    }

    if(i === 6){
      viewHeight = arg;
    }
});

page.viewportSize = { width: viewWidth, height:viewHeight };

if(cookieJson != null && cookieJson != ""){
    var cookieArray =  JSON.parse(cookieJson);
      for (var i = 0; i < cookieArray.length; i++) {
          var cookieObj = cookieArray[i];
          var cookie = {
            'name'     : cookieObj.name,
            'value'    : cookieObj.value,
            'path'     : cookieObj.path,
            'domain'   : cookieObj.domain,
            'expires'  : cookieObj.expires
          };
        phantom.addCookie(cookie);
    }
}

page.onError = function(msg, trace) {
    console.log("[Warning]This is page.onError");
    var msgStack = ['ERROR: ' + msg];
    if (trace && trace.length) {
        msgStack.push('TRACE:');
        trace.forEach(function(t) {
          msgStack.push(' -> ' + t.file + ': ' + t.line + (t.function ? ' (in function "' + t.function +'")' : ''));
        });
    }
    // console.error(msgStack.join('\n'));
};

phantom.onError = function(msg, trace) {
    console.log("[Warning]This is phantom.onError");
    var msgStack = ['PHANTOM ERROR: ' + msg];
    if (trace && trace.length) {
      msgStack.push('TRACE:');
      trace.forEach(function(t) {
        msgStack.push(' -> ' + (t.file || t.sourceURL) + ': ' + t.line + (t.function ? ' (in function ' + t.function +')' : ''));
      });
    }
      console.error(msgStack.join('\n'));
      phantom.exit(1);
};

page.onLoadFinished = function() {
   setTimeout(function(){
      if(type == "image"){
        page.render(fileName,{format: 'jpg', quality: '100'});
        console.log("render page finished: " + url);
      }
       
      if(type == "html"){
         fs.write(fileName, page.content, 'w');
         console.log("page html gen success: " + url);
      }

      phantom.exit();         
   },5000);
};

page.open(url, function (status) {
    if (status === "success") {
        console.log("load status: " + status);
    }else{
        phantom.exit();
    }
});
```

服务端通过Process类调用Linux中的shell脚本，运行截图或者页面渲染服务，然后把获取到的结果存储到文件存储系统中，返回文件存储的地址URL给业务方，业务方通过URL访问原始数据，减少网络数据传输。

```java
String type = "html";
String[] params =
                    new String[] {DiamondUtil.getPhantomJsPath(), DiamondUtil.getCaptureJsPath(), url,type , htmlName,
                        cookieJson};
                ShellUtil.executeShell(params, " capture url " + url);
                File htmlFile = new File(defaultFilePath(htmlName));
                if (checkFileEixst(htmlFile)) {
                    String htmlOssUrl = fileService.updateObjectToOss(htmlName, htmlFile);
                }
```

ShellUtil中的运行代码：

```java
public static void executeShell(String[] params, String message) throws Exception {
        Process process = null;
        BufferedReader stdBufferedReader = null;
        BufferedReader errorBufferedReader = null;
        try {
            ProcessBuilder pb = new ProcessBuilder(params);
            pb.directory(new File(DiamondUtil.getCaptureImagePath()));
            pb.redirectErrorStream(true);
            process = pb.start();
            stdBufferedReader = new BufferedReader(new InputStreamReader(process.getInputStream()));
            errorBufferedReader = new BufferedReader(new InputStreamReader(process.getErrorStream()));
            String line;
            while ((line = stdBufferedReader.readLine()) != null) {
                logger.info(message + ":" + line);
            }

            while ((line = errorBufferedReader.readLine()) != null) {
                logger.error(message + ":" + line);
            }
            process.waitFor();
        } catch (Exception e) {
            logger.error(message, e);
        } finally {
            if (stdBufferedReader != null) {
                stdBufferedReader.close();
            }
            if (errorBufferedReader != null) {
                stdBufferedReader.close();
            }
            if (process != null) {
                process.destroy();
            }

        }
    }
```

在截图服务中，首次出现了中文乱码，网上有同学说安装bitmap-fonts，但是并不好使，建议删除：


```shell
sudo yum remove bitmap-fonts bitmap-fonts-cjk
```

网页查看我们使用的字体内容：


```html
...{  ...  font-family: Helvetica, arial, "microsoft yahei", Monaco, sans-serif;  ...}
```

可以看到有 多种字体，最后的 sans-serif 默认对应是黑体，需要把这5种字体装到截图的 Linux 服务器，分别从osx和windows上copy出来，注意一下，osx 中的字体是 .ttc 和 .dfont 格式的，我们 可以借助 http://transfonter.org/ttc-unpack来转换为 Linux 支持的 .ttf 的格式。

把这些文件拷到 Linux 服务器上，然后调用 fc-cache 更新一下


```shell
sudo mkdir /usr/share/fonts/customsudo cp *.ttf  /usr/share/fonts/customfc-cache -fv
```

