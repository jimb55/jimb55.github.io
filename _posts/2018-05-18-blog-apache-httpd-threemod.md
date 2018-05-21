---
title: 'apache 三大模块'
date: 2018-05-18
tags:
  - linux
  - apache
toc: true
toc_label: "目录"
header:
  teaser: /assets/images/teaser/apache.png
---

## apache 三大模块
Prefork MPM模式：使用多个进程，每个进程只有一个线程，每个进程在某个确定的时间只能维持一个连接，稳定，内存开销较高；

![Image text](/assets/images/blogs/apache-httpd-threemod/prefork.jpg)    

处理链接队列请求，最大链接数配置
```
<IfModule prefork.c> 
MaxRequestsPerChild  4000 #每个进程能处理的最大请求数；
</IfModule>
```
 
![Image text](/assets/images/blogs/apache-httpd-threemod/prw.jpg)   

Worker MPM模式：使用多个进程，每个子进程包含多个线程，每个线程在某个确定的时间只能维持一个连接，内存占用量比较小，适合大并发、高流量的WEB服务器。Worker MPM缺点是一个线程崩溃，整个进程就会连同其任何线程一起挂掉

![Image text](/assets/images/blogs/apache-httpd-threemod/work.jpg)   

Event MPM模式：和worker工作模式很像，最大的区别是解决了在keep-alive场景下，长期被占用的线程的资源浪费问题，
在event MPM中，会有一个专门的线程来管理这些keep-alive线程，当有真实请求过来的时候，将请求传递给服务线程，
执行完毕后，又允许它释放，这样增强了在高并发场景下的请求处理能力。

![Image text](/assets/images/blogs/apache-httpd-threemod/event.jpg) 

> keep-alive 使用keep-alive可以改善这种状态，即在一次TCP连接中可以持续发送多份数据而不会断开连接。也可称为长链接

> Event MPM 不支持 https，一般的公司生产环境都不用！

参考资料：http://blog.jobbole.com/91920/

## 更改 Server MPM 模式

查看当前MPM模式
```
[root@localhost apache2]# /usr/local/apache2/bin/apachectl -V
Server version: Apache/2.4.33 (Unix)
Server built:   May 18 2018 12:00:10
Server's Module Magic Number: 20120211:76
Server loaded:  APR 1.6.3, APR-UTIL 1.6.1
Compiled using: APR 1.4.8, APR-UTIL 1.5.2
Architecture:   64-bit
Server MPM:     event
  threaded:     yes (fixed thread count)
    forked:     yes (variable process count)
```

更换模式只能在预编译时指定，如
```
--with-mpm=prefork 
--with-mpm=worker
--with-mpm=event
```
也可以全部都安装，到配置文件中切换
```
--enable-mpms-shared=all

# 配置文件
#LoadModule mpm_prefork_module modules/mod_mpm_prefork.so
#LoadModule mpm_worker_module modules/mod_mpm_worker.so
#LoadModule mpm_event_module modules/mod_mpm_event.so
```

例如切换worker模式

```
[root@localhost httpd-2.4.33]# /usr/local/apache2/bin/apachectl stop
[root@localhost httpd-2.4.33]# ./configure --prefix=/usr/local/apache2/ --enable-rewrite --enable-so  --with-included-apr --with-mpm=pworker
[root@localhost httpd-2.4.33]# make
[root@localhost httpd-2.4.33]# make install
[root@localhost httpd-2.4.33]# /usr/local/apache2/bin/apachectl start
[root@localhost httpd-2.4.33]# /usr/local/apache2/bin/apachectl -V
Server version: Apache/2.4.33 (Unix)
Server built:   May 18 2018 15:30:41
Server's Module Magic Number: 20120211:76
Server loaded:  APR 1.6.3, APR-UTIL 1.6.1
Compiled using: APR 1.4.8, APR-UTIL 1.5.2
Architecture:   64-bit
Server MPM:     worker
  threaded:     yes (fixed thread count)
    forked:     yes (variable process count)
```
切换为 worker模式成功 

全部安装

```
[root@localhost httpd-2.4.33]# /usr/local/apache2/bin/apachectl stop
[root@localhost httpd-2.4.33]# ./configure --prefix=/usr/local/apache2/ --enable-rewrite --enable-so  --with-included-apr --enable-mpms-shared=all
[root@localhost httpd-2.4.33]# make
[root@localhost httpd-2.4.33]# make install

[root@localhost httpd-2.4.33]# /usr/local/apache2/bin/apachectl start
AH00534: httpd: Configuration error: No MPM loaded. #得在配置文件打开一个MPM

[root@localhost httpd-2.4.33]# cd /usr/local/apache2
[root@localhost apache2]# vi conf/httpd.conf 
...
#LoadModule speling_module modules/mod_speling.so
#LoadModule userdir_module modules/mod_userdir.so
LoadModule alias_module modules/mod_alias.so
#LoadModule rewrite_module modules/mod_rewrite.so

# server MPM 打开了 prefork 
# 没有下面代码自行添加
LoadModule mpm_prefork_module modules/mod_mpm_prefork.so 
#LoadModule mpm_worker_module modules/mod_mpm_worker.so
#LoadModule mpm_event_module modules/mod_mpm_event.so
...

[root@localhost apache2]# /usr/local/apache2/bin/apachectl start
[root@localhost apache2]# /usr/local/apache2/bin/apachectl -V
Server version: Apache/2.4.33 (Unix)
Server built:   May 18 2018 15:34:05
Server's Module Magic Number: 20120211:76
Server loaded:  APR 1.6.3, APR-UTIL 1.6.1
Compiled using: APR 1.4.8, APR-UTIL 1.5.2
Architecture:   64-bit
Server MPM:     prefork
  threaded:     no
    forked:     yes (variable process count)
```

