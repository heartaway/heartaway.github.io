---
layout: post
title: Github Pages Build Blog
categories: Tool
description:
keywords: github.pages
date: 2017-05-18
---
在github上新建一个项目，项目名称为““projectName.github.io”” ,其中projectName需要替换为你在github上的账号名称。

### 本地环境搭建：

切换ruby空间：

```shell
gem sources --remove https://ruby.taobao.org/
```

更新gem:

```shell
sudo gem update --system
```

安装ruby 2.0 以上版本，查看ruby版本方式： 

```shell
 ruby -v 
```

ruby 的管理工具： rvm
安装2.1.4版本：

```shell
rvm install 2.0.0 
```

设置ruby 2.0.0 版本为默认版本：

```shell
rvm use 2.0.0 --default
```

删除ruby1.9.x版本：

```shell
rvm  remove  1.9.x
```

删除本地用户目录下的ruby


#### 运行方式一：直接使用jekyll：

安装server：

```shell
gem install -n /usr/local/bin/ jekyll
```

运行 jekyll :

```
jekyll serve --safe --watch
```

#### 运行方式二：使用bundle来运行jekyll：

安装bundle：

```shell
gem install -n /usr/local/bin/ bundle
```

在根目录下新建名为：Gemfile 的文件，内容为：

```shell
source 'https://rubygems.org'gem 'github-pages' , group::jekyll_plugins
```

然后执行：

```shell
bundle install
bundle update
```


运行 jekyll :

```shell
bundle exec jekyll serve
```

运行时出现端口异常：

```shell
jekyll 3.4.3 | Error:  Address already in use - bind(2)
```

解决方式：

```shell
sudo lsof -wni tcp:4000 | awk '{print $2}'|xargs kill -9
```

### 模板选择：
模板访问地址： http://jekyllthemes.org/， 选择合适的模板，Clone下来后，直接把模板中的内容拷贝到项目工程根目录中即可。

通过修改_config.yml 来配置博客内容；

### 域名绑定：
1. 在工程根目录中新建文件CNAME，内容为新域名地址；
2. 在github项目工程的setting中，找到Github Pages ,有一个Custom domain，填入你的域名；
3. 或者通过你的域名管理，直接设置CNAME，指向 xxx.github.io;

### 运行时异常：
一段时间后，出现一下错误：

```shell
GitHub Metadata: No GitHub API authentication could be found
```
这是因为无法获取到Github的API授权信息，我们可以到https://github.com/settings/profile 中找到 `Person access tokens` , 生成一个public_repo权限的token。
然后在个人本地的`~.bash_profile`中加入以下代码：

```shell
export JEKYLL_GITHUB_TOKEN='xxxxx'
```
然后在你的启动脚本中加入（start.sh）：

```shell
source ~/.bash_profile
export JEKYLL_GITHUB_TOKEN=$JEKYLL_GITHUB_TOKEN
nohup bundle exec jekyll serve > nohup.out  2>&1 &
```

注意：在运行了阿里郎的网络加速后，本地环境localhost:4000无法访问，必须关闭阿里郎网络加速；




参考：https://help.github.com/articles/setting-up-your-github-pages-site-locally-with-jekyll/

