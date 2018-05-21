---
title: 'apache 编译安装'
date: 2018-05-18
tags:
  - linux
  - apache
toc: true
toc_label: "目录"
header:
  teaser: /assets/images/teaser/apache.png
---

# apache 编译安装

## apache 下载
apache 官网 打开 project 栏目，找到 httpd（或Apache HTTP Server）
或者快捷链接 [apache server download](http://httpd.apache.org/download.cgi)

```
[root@localhost local]# wget http://mirrors.shu.edu.cn/apache//httpd/httpd-2.4.33.tar.gz
--2018-05-18 09:32:46--  http://mirrors.shu.edu.cn/apache//httpd/httpd-2.4.33.tar.gz
正在解析主机 mirrors.shu.edu.cn (mirrors.shu.edu.cn)... 202.121.199.235
正在连接 mirrors.shu.edu.cn (mirrors.shu.edu.cn)|202.121.199.235|:80... 已连接。
已发出 HTTP 请求，正在等待回应... 200 OK
长度：9076901 (8.7M) [application/x-gzip]
正在保存至: “httpd-2.4.33.tar.gz”

100%[================================================================================================================================================>] 9,076,901    895KB/s 用时 10s    

2018-05-18 09:32:56 (863 KB/s) - 已保存 “httpd-2.4.33.tar.gz” [9076901/9076901])
```

## apache 安装
```
[root@localhost local]# tar -zxvf httpd-2.4.33.tar.gz 
[root@localhost local]# cd httpd-2.4.33
[root@localhost local]# yum install apr apr-devel apr-util apr-util-devel -y
```
**apr**:
Apache portable Run-time libraries，Apache可移植运行库，主要为上层的应用程序提供一个可以跨越多操作系统平台使用的底层支持接口库
在早期的Apache版本中，应用程序本身必须能够处理各种具体操作系统平台的细节，并针对不同的平台调用不同的处理函数。随着Apache的进一步开发，Apache组织决定将这些通用的函数独立出来并发展成为一个新的项目。这样，APR的开发就从Apache中独立出来，Apache仅仅是使用APR而已。

apr中包含了一些通用的开发组件，包括mmap，DSO等等
apr-util该目录中也是包含了一些常用的开发组件。这些组件与apr目录下的相比，它们与apache的关系更加密切一些。比如存储段和存储段组，加密等等。
apr-iconv包中的文件主要用于实现iconv编码。目前的大部分编码转换过程都是与本地编码相关的。

预编译

```
[root@localhost httpd-2.4.33]# mkdir -p /usr/local/apache2
[root@localhost local]#  ./configure --prefix=/usr/local/apache2/ --enable-rewrite --enable-so
```
 - enable-rewrite 启用rewrite规则,开启重写功能
 - enable-so 启用动态加载库,让apache核心装载DSO

> ps : DSO是Dynamic SharedObjects（动态共享目标）的缩写，它是现代Unix派生出来的操作系统都存在着的一种动态连接机制
> Apache就是使用DSO来在运行时加载它的模块,参考资料 http://blog.chinaunix.net/uid-20773865-id-113909.html

编译&安装
```
[root@localhost local]# make # 编译
[root@localhost local]# make install # 安装
```

启动Apache

```
[root@localhost httpd-2.4.33]#  /usr/local/apache2/bin/apachectl start
[root@localhost apache2]# ps -ef | grep apache
root      47325      1  0 13:55 ?        00:00:00 /usr/local/apache2//bin/httpd -k start
daemon    47326  47325  0 13:55 ?        00:00:00 /usr/local/apache2//bin/httpd -k start
daemon    47327  47325  0 13:55 ?        00:00:00 /usr/local/apache2//bin/httpd -k start
daemon    47328  47325  0 13:55 ?        00:00:00 /usr/local/apache2//bin/httpd -k start
root      47411   3224  0 13:57 pts/0    00:00:00 grep --color=auto apache
```



## 可能遇到的问题

### 预编译环境检测问题
1 缺失 gcc 库

```
checking for gcc... gcc
checking whether the C compiler works... no
configure: error: in `/root/local/httpd-2.4.33':
configure: error: C compiler cannot create executables
See `config.log' for more details
```

解决方法

```
# yum install gcc libc6-dev
```

2 缺失 pcre 库

```
checking for pcre-config... false
configure: error: pcre-config for libpcre not found. PCRE is required and available from http://pcre.org/
```

解决方法

```
yum install pcre pcre-devel
```

### 编译问题

1 找不到 apr 位置问题

确保你系统已经安装 apr apr-util ,但make 时仍然报下面的错误

```
.33/include -I/usr/include/apr-1 -I/root/local/httpd-2.4.33/modules/aaa -I/root/local/httpd-2.4.33/modules/cache -I/root/local/httpd-2.4.33/modules/core -I/root/local/httpd-2.4.33/modules/database -I/root/local/httpd-2.4.33/modules/filters -I/root/local/httpd-2.4.33/modules/ldap -I/root/local/httpd-2.4.33/modules/loggers -I/root/local/httpd-2.4.33/modules/lua -I/root/local/httpd-2.4.33/modules/proxy -I/root/local/httpd-2.4.33/modules/session -I/root/local/httpd-2.4.33/modules/ssl -I/root/local/httpd-2.4.33/modules/test -I/root/local/httpd-2.4.33/server -I/root/local/httpd-2.4.33/modules/md -I/root/local/httpd-2.4.33/modules/arch/unix -I/root/local/httpd-2.4.33/modules/dav/main -I/root/local/httpd-2.4.33/modules/generators -I/root/local/httpd-2.4.33/modules/mappers -prefer-pic -c mod_proxy_balancer.c && touch mod_proxy_balancer.slo
mod_proxy_balancer.c:25:24: fatal error: **apr_escape.h**: No such file or directory
 #include "apr_escape.h"
                        ^
```

我安装的 Apache 是 2.4.33 版本，不知道为何找不到apr 地址，官方上也没找到满意的解决办法。
但 yum 安装的 apr 和 apr-util没效，可以尝试官方提供的手动依赖apr 的方法

```
APR and APR-Util
Make sure you have APR and APR-Util already installed on your system. 
If you don't, or prefer to not use the system-provided versions, 
download the latest versions of both APR and APR-Util from Apache APR, 
unpack them into /httpd_source_tree_root/srclib/apr and /httpd_source_tree_root/srclib/apr-util 
(be sure the directory names do not have version numbers; 
for example, the APR distribution must be under /httpd_source_tree_root/srclib/apr/) 
and use ./configure's --with-included-apr option. On some platforms, you may have to 
install the corresponding -dev packages to allow httpd to build against your installed copy 
of APR and APR-Util.
```

这说法也不全对，下载的 apr 包得放到源码安装目录下的 srclib 文件夹，不然得启动 `--with-included-apr ` 的同时还得指定`--with-apr=PATH`

下载apr 和 apr-util
![Image text](/assets/images/blogs/apache-httpd-install/3dowapr.png)      

```
[root@localhost srclib]# pwd
/root/local/httpd-2.4.33/srclib
[root@localhost srclib]# wget http://mirrors.shu.edu.cn/apache//apr/apr-1.6.3.tar.gz
...
[root@localhost srclib]# wget http://mirrors.shu.edu.cn/apache//apr/apr-util-1.6.1.tar.gz
...

[root@localhost srclib]# tar -zxvf apr-1.6.3.tar.gz
[root@localhost srclib]# tar -zxvf apr-util-1.6.1.tar.gz
[root@localhost srclib]# mv apr-1.6.3 apr
[root@localhost srclib]# mv apr-util-1.6.1 apr-util   
[root@localhost srclib]# rm apr-1.6.3.tar.gz
[root@localhost srclib]# rm apr-util-1.6.1.tar.gz
[root@localhost srclib]# ls -al
总用量 20
drwxr-sr-x.  4 root 40   81 5月  18 11:59 .
drwxr-sr-x. 13 root 40 4096 5月  18 13:42 ..
drwxr-sr-x. 28 root 40 4096 5月  18 11:59 apr
drwxr-sr-x. 21 root 40 4096 5月  18 12:00 apr-util
-rw-r--r--.  1 root 40    0 5月  18 11:59 .deps
-rw-r--r--.  1 root 40  342 5月  18 11:59 Makefile
-rw-r--r--.  1 root 40  121 2月  11 2005 Makefile.in
[root@localhost srclib]# cd ../
[root@localhost httpd-2.4.33]# 
```

重新执行预编译,但命令需要改变
```
[root@localhost httpd-2.4.33]#  ./configure --prefix=/usr/local/apache2/ --enable-rewrite --enable-so  --with-included-apr
[root@localhost httpd-2.4.33]#  make
[root@localhost httpd-2.4.33]#  make install
```

### 服务开启问题

1没有配置ServerName 问题

```
[root@localhost httpd-2.4.33]#  /usr/local/apache2/bin/apachectl start
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using localhost.localdomain. Set the 'ServerName' directive globally to suppress this message
```

编辑 httpd 配置文件
```
[root@localhost apache2]# pwd
/usr/local/apache2
[root@localhost apache2]# vi conf/httpd.conf
```

修改配置文件

```
#
# ServerName gives the name and port that the server uses to identify itself.
# This can often be determined automatically, but we recommend you specify
# it explicitly to prevent problems during startup.
#
# If your host doesn't have a registered DNS name, enter its IP address here.
#
#ServerName www.example.com:80
ServerName localhost  #此处打开，配置你的域名
```

重启服务器,便不会再报上面的错了

```
[root@localhost apache2]# /usr/local/apache2/bin/apachectl stop
[root@localhost apache2]# ps -ef | grep apache
root      47439   3224  0 14:02 pts/0    00:00:00 grep --color=auto apache
[root@localhost apache2]# /usr/local/apache2/bin/apachectl start
[root@localhost apache2]# ps -ef | grep apache                  
root      47443      1  0 14:02 ?        00:00:00 /usr/local/apache2//bin/httpd -k start
daemon    47444  47443  0 14:02 ?        00:00:00 /usr/local/apache2//bin/httpd -k start
daemon    47445  47443  0 14:02 ?        00:00:00 /usr/local/apache2//bin/httpd -k start
daemon    47446  47443  0 14:02 ?        00:00:00 /usr/local/apache2//bin/httpd -k start
root      47529   3224  0 14:02 pts/0    00:00:00 grep --color=auto apache
```

2 服务启动后不能访问问题
一般情况下是 如果服务正常运行，那么可能是防火墙问题，关闭后试试
```
[root@localhost apache2]# firewall-cmd --zone=public --add-port=80/tcp --permanent
success
[root@localhost apache2]# firewall-cmd --reload   #重启防火墙
success
```
 - --zone #作用域
 - --add-port=80/tcp #添加端口，格式为：端口/通讯协议
 - --permanent #永久生效，没有此参数重启后失效