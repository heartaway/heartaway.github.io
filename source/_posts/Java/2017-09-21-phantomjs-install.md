---
layout: post
title: Linux phantomJs install course
categories: Java
description:
keywords: phantomJS
date: 2017-09-21
---

> 背景：
> 我们在进行系统报告研发时，需要对报告结果进行定时邮件，所以需要把HTML内容自动转换为邮件内容，通常我们在发送邮件内容的时候内容是模板化的，模板基本不怎么改变，然而我们的报告却是由无数个子模块组成，这些子模块可以只有组合成为任意报告，每一个模块中的内容均为用户自定义，如果我们要实现通用化报告邮件化，那就需要把渲染后的报告转换为图片或者HTML格式，headless方案中常用的有phantomJS和Headless Chrome，phantomJS提供了截图等功能，所以称为我们的首选。

#### 遇到的问题：
我们生产环境大部分机器操作系统为5u，默认的glibc版本最高到2.5。然而 PhantomJS官网中明确指出Linux 64-bit操作系统下，需要依赖GLIBCXX_3.4.9 和 GLIBC_2.7/GLIBC_2.9/GLIBC_2.10。
![](/images/posts/20170921/15059814992587.jpg)

但是在我们生产环境中：

```shell
strings /lib64/libc.so.6 |grep GLIBC
strings /usr/lib64/libstdc++.so.6 |grep GLIBCXX
```

![](/images/posts/20170921/15059815133485.jpg)

所以需要安装：GLIBCXX_3.4.9  GLIBC_2.7



### 一、安装phantoomjs
#### 1. 安装node：
```java
sudo wget  http://xxxxx/node-v6.11.2-linux-x64.tar.xz
sudo xz -d node-v6.11.2-linux-x64.tar.xz
sudo tar xvf node-v6.11.2-linux-x64.tar

sudo ln -s /home/admin/phantomjs/node-v6.11.2-linux-x64/bin/node /usr/local/bin/node
sudo ln -s /home/admin/phantomjs/node-v6.11.2-linux-x64/bin/npm /usr/local/bin/npm

sudo vi /etc/profile
PATH=$PATH:/home/admin/phantomjs/node-v6.11.2-linux-x64/bin

node -v
```


#### 2. 安装npm：
```shell
curl -vvvL https://npmjs.com/install.sh >/dev/null
```

#### 3. 安装phantomJS：
```shell
sudo wget http://xxxx/phantomjs-2.1.1-linux-x86_64.tar.bz2
sudo tar -jxvf phantomjs-2.1.1-linux-x86_64.tar.bz2
sudo ln -s /home/admin/phantomjs/phantomjs-2.1.1-linux-x86_64/bin/phantomjs /usr/local/bin/phantomjs

phantomjs --version
```
![](/images/posts/20170921/15059819937015.jpg)
我们的系统中没有这些动态库，有联两种办法，第一自己安装这些版本，第二升级操作系统到7u，我们先尝试使用第一种方法。

### 二、安装GCC
因为部分版本的Glibc的安装需要高版本的GCC，默认的GCC版本为4.1.2，导致不能编译glibc的2.10以上版本，所以必须升级，gcc>=4.3.3。

```shell
cd  /home/admin/gcc/gcc-build
sudo yum install libstdc++-devel.i686
sudo yum install  libstdc++-devel.x86_64
sudo ../gcc-4.8.2/configure --prefix=/usr/local/gcc-4.8.2  --with-gmp=/usr/local/gmp-6.0.0 --with-mpfr=/usr/local/mpfr-3.1.2 --with-mpc=/usr/local/mpc-1.0.1  --with-java-home=/opt/taobao/java --disable-multilib  --disable-shared --enable-threads=posix --disable-checking  --enable-languages=all  --enable-static --enable-shared=libstdc++,libgcc_eh
```

或者：

```shell
sudo ../gcc-4.8.2/configure --prefix=/usr/local/gcc-4.8.2 --enable-threads=posix  --with-gmp=/usr/local/gmp-6.0.0 --with-mpfr=/usr/local/mpfr-3.1.2 --with-mpc=/usr/local/mpc-1.0.1  --mandir=/usr/share/man --infodir=/usr/share/info --enable-shared --disable-multilib --enable-checking=release --with-system-zlib --enable-__cxa_atexit --disable-libunwind-exceptions --enable-libgcj-multifile --enable-languages=c,c++,objc,obj-c++,java --disable-dssi --disable-plugin --with-java-home=/opt/taobao/java/jre --with-cpu=generic

sudo make
sudo make install 

#配置环境变量：
sudo vi /etc/profile
export PATH=$PATH:/usr/local/gcc-4.8.2
source /etc/profile

sudo rm /usr/bin/gcc    //删除旧的软连接  
sudo ln -s /usr/local/gcc-4.8.2/bin/gcc /usr/bin/gcc //使新版本建立软连接 
```

安装完毕后，查看GCC版本：
```shell
$gcc -v
Using built-in specs.
COLLECT_GCC=gcc
COLLECT_LTO_WRAPPER=/usr/local/gcc-4.8.2/libexec/gcc/x86_64-unknown-linux-gnu/4.8.2/lto-wrapper
Target: x86_64-unknown-linux-gnu
Configured with: ../gcc-4.8.2/configure --prefix=/usr/local/gcc-4.8.2 --with-gmp=/usr/local/gmp-6.0.0 --with-mpfr=/usr/local/mpfr-3.1.2 --with-mpc=/usr/local/mpc-1.0.1 --with-java-home=/opt/taobao/java --disable-multilib --disable-shared --enable-threads=posix --disable-checking --enable-static --enable-shared=libstdc++
Thread model: posix
gcc version 4.8.2 (GCC)
```

##### 出现错误1：

```shell
configure: error: cannot compute suffix of object files: cannot compile See `config.log' for more details
```
查看config.log，发现错误：

```shell
/home/admin/gcc/gcc-build/./gcc/cc1: error while loading shared libraries: libmpc.so.3: cannot open shared object file: No such file or directory
```
说明是环境变量设置未生效，检查环境变量&ldconfig相关设置。

##### 出现错误2：

```shell
/usr/local/include/gnu/stubs.h:7:27: fatal error: gnu/stubs-32.h: No such file or directory
compilation terminated.
```
安装：
查看可用的版本：yum list available glibc-devel
sudo yum install glibc-devel
sudo yum install libstdc++-devel.i686

##### 出现错误3：

```shell
/usr/lib/gcc/x86_64-redhat-linux/4.1.2/../../../../include/c++/4.1.2/iosfwd:44:28: error: bits/c++config.h: No such file or directory
/usr/lib/gcc/x86_64-redhat-linux/4.1.2/../../../../include/c++/4.1.2/iosfwd:45:29: error: bits/c++locale.h: No such file or directory
/usr/lib/gcc/x86_64-redhat-linux/4.1.2/../../../../include/c++/4.1.2/iosfwd:46:25: error: bits/c++io.h: No such file or directory
```
解决方案：

```shell
sudo yum install  libstdc++-devel.x86_64
```

##### 出现错误4：

```shell
lib/libgmp.so: could not read symbols: File in wrong format collect2: error: ld returned 1 exit status
/home/admin/gcc/gcc-build/x86_64-unknown-linux-gnu/32/libjava/classpath/native/jni/java-math'
```
解决方案：


```shell
sudo ../gcc-4.8.2/configure --prefix=/usr/local/gcc-4.8.2 --enable-threads=posix --disable-checking --with-gmp=/usr/local/gmp-6.0.0 --with-mpfr=/usr/local/mpfr-3.1.2 --with-mpc=/usr/local/mpc-1.0.1 --enable-shared --enable-languages=all  --enable-libgomp --enable-lto --enable-tls --with-fpmath=sse --disable-multilib --build=x86_64-redhat-linux    --with-java-home=/opt/taobao/java
```

##### 出现错误5：

```shell
Error: unrecognized symbol type "gnu_unique_object"
```
解决方案：移除 configure参数中的 gnu_unique_object 配置；

##### 出现错误6：

```shell
configure: error: libXtst not found, required by java.awt.Robot
```

解决方案：移除 configure参数中的 java.awt 配置；

##### 出现错误7：
libtool: compile: not configured to build any kind of library
解决方案：
sudo ../gcc-4.8.2/configure --prefix=/usr/local/gcc-4.8.2  --with-gmp=/usr/local/gmp-6.0.0 --with-mpfr=/usr/local/mpfr-3.1.2 --with-mpc=/usr/local/mpc-1.0.1  --with-java-home=/opt/taobao/java --disable-multilib --disable-shared --enable-threads=posix --disable-checking  --enable-languages=all  --enable-static --enable-shared=libstdc++,libgcc_eh

##### 出现错误8：
cc1plus: error: unrecognized command line option "-Wno-narrowing"
解决方案：目前没有查询到好的解决方案；

##### 出现错误9：
/usr/include/string.h:550:18: error: unknown type name ‘__locale_t’
解决方案：暂无

##### 出现错误10：
/usr/lib64/libstdc++.so.6: undefined symbol: _ZNSt7num_getIcSt19istreambuf_iteratorIcSt11char_traitsIcEEE2idE, version GLIBCXX_3.4


### 三、安装GlibC
查看当前系统Glibc的版本：

```shell
strings /lib64/libc.so.6 |grep GLIBC
```

编译glibc：

```shell
sudo mkdir glic
cd glic
sudo wget http://ftp.gnu.org/pub/gnu/glibc/glibc-2.7.tar.gz
sudo wget http://ftp.gnu.org/pub/gnu/glibc/glibc-2.9.tar.gz
//sudo wget http://ftp.gnu.org/pub/gnu/glibc/glibc-2.10.1.tar.gz
sudo wget http://ftp.gnu.org/pub/gnu/glibc/glibc-2.14.tar.gz

sudo tar zxvf glibc-2.7.tar.gz
sudo tar zxvf glibc-2.9.tar.gz
sudo tar zxvf glibc-2.10.1.tar.gz
sudo tar zxvf glibc-2.14.tar.gz

sudo mkdir glibc-build-2.7
sudo mkdir glibc-build-2.9
sudo mkdir glibc-build-2.10.1
sudo mkdir glibc-build-2.14

cd glibc-build-2.7

sudo ../glibc-2.7/configure
```

遇到错误：
configure: error: no acceptable C compiler found in $PATH
安装gcc套件(rpm -qa |grep gcc  可以看到目前的版本时4.1.2，注意此版本太老，导致不能编译glibc的2.10以上版本，所以必须升级，gcc>=4.3.3)：


遇到错误：

```shell
*** On GNU/Linux systems the GNU C Library should not be installed into
*** /usr/local since this might make your system totally unusable.
*** We strongly advise to use a different prefix.  For details read the FAQ.
```

INSTALL文件摘录如下：

> `configure' takes many options, but the only one that is usually
> mandatory is `--prefix'.  This option tells `configure' where you want
> glibc installed.  This defaults to `/usr/local', but the normal setting
> to install as the standard system library is `--prefix=/usr' for
> GNU/linux systems and `--prefix=' (an empty prefix) for GNU/Hurd
> systems.

关于安装目录的选择:


```shell
--prefix=PREFIX
```

安装目录，默认为 /usr/local
Linux文件系统标准要求基本库必须位于 /lib 目录并且必须与根目录在同一个分区上，但是 /usr 可以在其他分区甚至是其他磁盘上。因此，如果指定 --prefix=/usr ，那么基本库部分将自动安装到 /lib 目录下，而非基本库部分则会自动安装到 /usr/lib 目录中。但是如果保持默认值或指定其他目录，那么所有组件都间被安装到PREFIX目录下。


```shell
sudo ../glibc-2.7/configure --prefix=/usr --disable-multi-arch
sudo ../glibc-2.9/configure --prefix=/usr --disable-multi-arch
sudo ../glibc-2.14/configure --prefix=/usr --disable-multi-arch
```


在每个build目录下，执行glibc的配置：

```shell
sudo ../glibc-2.7/configure --prefix=/usr --disable-multi-arch
sudo ../glibc-2.9/configure --prefix=/usr --disable-multi-arch
sudo ../glibc-2.10.1/configure --prefix=/usr --disable-multi-arch
```

分别进入每个build目录，执行命令(在 glibc-build-xx 目录下)：

```shell
sudo make all && make install
```

如果直接执行（sudo make install）出现错误提示：

```shell
make[2]: *** No rule to make target `/home/admin/glibc-build/dlfcn/libdl.so.2', needed by `/home/admin/glibc-build/elf/sprof'.  Stop.
make[2]: Leaving directory `/home/admin/glibc-2.7/elf'
make[1]: *** [elf/subdir_install] Error 2
make[1]: Leaving directory `/home/admin/glibc-2.7'
make: *** [install] Error 2
```

解决方案(先执行make all， 再执行 make install);

安装完毕后，出现提示：


```shell
Your new glibc installation seems to be ok.
make[1]: Leaving directory `/home/admin/glibc-2.7'
```


##### 常见错误信息1：

```shell
/lib/modules/2.6.32-220.23.2.ali927.el5.x86_64/build/include/linux/swab.h:6:22: fatal error: asm/swab.h: No such file or directory
 #include <asm/swab.h>
```

解决办法：


```shell
sudo touch /lib/modules/2.6.32-220.23.2.ali927.el5.x86_64/build/include/asm-x86/swab.h
```

参考：https://bbs.archlinux.org/viewtopic.php?id=70488

##### 常见错误信息2：
configure: error: assembler too old, .cfi_personality support missing
解决办法：google说gcc版本太老。

##### 常见错误信息3：

```shell
/usr/bin/ld: cannot find -lgcc_eh
```

原因是： glibc在安装的时候，需要使用gcc_eh（--enable-shared），但是我们在安装gcc时，添加了--disable-shared，禁用了shared功能，所以导致此库找不到；
测试gcc_eh是否安装：sudo gcc -lgcc_eh --verbose

解决办法：

```shell
sudo find / -name libgcc.a
sudo cp libgcc.a libgcc_eh.a
```

参考：
http://www.linuxfromscratch.org/clfs/view/clfs-2.0/arm/cross-tools/glibc.html
https://gcc.gnu.org/ml/gcc-patches/2005-02/msg00532.html
http://lists.linuxfromscratch.org/pipermail/lfs-support/2012-December/044174.html

##### 常见错误信息4：

```shell
 Can't open configuration file /usr/etc/ld.so.conf: No such file or directory
```

参考：
http://www.math.ias.edu/~tarzadon/pages/posts/install-opam-on-springdalerhelcentossl-5.x-111.php


##### 常见错误信息5：

```shell
/build/include/linux/capability.h:73: error: expected specifier-qualifier-list before ‘__le32’
```

解决方案：
参考：
https://github.com/spotify/linux/blob/master/include/linux/capability.h
sudo yum install openssl-devel gnutls-devel libcap-devel
查看版本：
strings /lib64/libc.so.6 |grep GLIBC


### 四、安装GLIBCXX
查看版本：strings /usr/lib64/libstdc++.so.6 |grep GLIBCXX
查看：ls -l /usr/lib64/libstdc++.so.6
lrwxrwxrwx 1 root root 18 Aug 14  2015 /usr/lib64/libstdc++.so.6 -> libstdc++.so.6.0.8

自己编译gcc的过程太繁琐，需要依赖非常多的文件，直接下载高版本下的文件进行替换：


```shell
sudo wget http://xxx/libstdc%2B%2B.so.6.0.13
sudo cp libstdc++.so.6.0.13 /usr/lib
sudo cp libstdc++.so.6.0.13 /usr/lib64

cd /usr/lib
sudo ln -sf libstdc++.so.6.0.13  libstdc++.so.6

cd /usr/lib64
sudo ln -sf libstdc++.so.6.0.13  libstdc++.so.6
```

再次执行


```shell
strings /usr/lib64/libstdc++.so.6 |grep GLIBCXX
```

发现需要的安装包已经安装完毕；


### 五、MAC下安装phantomJS：
下载文件到本地目录；
添加环境变量：

```shell
vi .bash_profile
export PATH=$PATH:/Users/xinyuan/soft/phantomjs-2.5.0-beta-macos/bin
source  .bash_profile
```

安装webp：

```shell
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)" < /dev/null 2> /dev/null
brew install webp
```

建立软连：

```shell
ln -s  /usr/local/opt/webp/lib/libwebp.7.dylib /usr/local/opt/webp/lib/libwebp.6.dylib
```

查看版本：

```shell
phantomjs -v
2.5.0-development
```

### 六、总结：
整个OS的环境安装由于对linux底层不熟悉，参数配置也没有详细查看文档，导致安装过程出现了非常多的意外，之前使用的操作系统是5u，gcc版本为4.1.2，依赖的glibc、glibxx、gmp、mpfr、mpc库版本都比较低，但是phantomjs（JS截图）需要依赖其高版本库，为了升级响应的库，通过下载源码，make、install ，出现了不下于30个错误，每次解决一个错误后续流程中又出现了新的错误，费劲周折把所有依赖的库都装好后，运行phantomjs还是出错，怀疑是缺少了一些动态库；  后来还是通过升级操作系统，解决了此问题，升级操作系统完成docker化文件配置，总共才花费了不到一天时间。反思：遇到问题，首选需要评估一下解决此问题的最有效手段，而不是闷头上去搞，要看我们从中能得到什么，在linux操作系统上安装一坨软件并不能证明我们有什么能力，所以这件事情就不应该自己去尝试做，投入产出比是做事情非常重要的一个衡量手段。

### 七、参考链接：
http://blog.csdn.net/tsaiyong_ahnselina/article/details/21552485
http://siliconcali.com/2012/10/glibcxx_3-4-9-not-found-in-red-hatfedoracentos-linux/
http://blog.csdn.net/tengdazhang770960436/article/details/41348035
http://www.cnblogs.com/LitLeo/p/3534196.html
http://blog.csdn.net/wtfmonking/article/details/17577925
http://www.cnblogs.com/coolulu/p/4124803.html



