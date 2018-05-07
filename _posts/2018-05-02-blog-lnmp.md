---
title: 'lnmp 基础安装'
date: 2015-08-14
permalink: /posts/2012/08/blog-post-4/
tags:
  - php
  - linux
  - mysql
  - nginx
toc: true
toc_label: "目录"
teaser: /assets/images/tmp2.jpg
---

# linux 基础
安装PHP7.0 nginx mysql

### 安装 PHP7
登录 php.net/ 打开downloads 页面
选个最新的（php.net 上的php版本都为linux 源码包，需要window 的到window.php.net/ 上下载，另需要旧版本在download 的 Old archives 页面下寻找）

下载php7 源码包
```
wget http://cn2.php.net/distributions/php-7.2.3.tar.gz
```
>ps：假如用window查找的资源，可直接在浏览器上复制其下载地址到linux 上下载！

解压下载包
```
tar -zxvf  php-7.2.3.tar.gz
```

到解压目录
```
cd php-7.2.3
```
会发现一大堆文件，这些都是源码，主要是./configure 文件 网上教程都是 `./configure && make && make install` 

打开配置帮助
```
./configure --help
```
会打印一大堆配置要求，但什么curl giz pdo 先不管，主要是三处
- --prefix=/... 配置安装路径
- --with-config-file-path=/... 这个是php.ini存放路径，只能现在配置，编译安装后就不能指定了，要重新编译，所以这里得先指定
- --enable-fpm  安装 php-fpm,nginx需要的fast-cig需指定php-fpm ,也就是传统的php-fpm 9000端口

如例子
```
./configure --prefix=/usr/local/es/php7 --with-config-file-path=/usr/local/es/php7/etc/ --enable-fpm
```
成功如下
``` linux
Generating files
configure: creating ./config.status
creating main/internal_functions.c
creating main/internal_functions_cli.c
+--------------------------------------------------------------------+
| License:                                                           |
| This software is subject to the PHP License, available in this     |
| distribution in the file LICENSE.  By continuing this installation |
| process, you are bound by the terms of this license agreement.     |
| If you do not agree with the terms of this license, you must abort |
| the installation process at this point.                            |
+--------------------------------------------------------------------+

Thank you for using PHP.
```
出现以上内容表示安装成功。
但经常会出现各种问题，最常见的是缺少环境
如下：
```
configure error xml2-config not found. please check your libxml2 installation
```
就是输出了一大串，然后在结尾（差不多结尾的地方）出现这样一个东东，基本上就是安装失败了。
如字面意思，就是缺少某个库 `libxml2` 
用 yum 直接补上缺失的库就行（这些库基本上yum 都可以装，不用自己去找下载地址和配置）
```linux
yum install libxml2
yum install libxml2-devel
```
把上面报错的包名（就是 please check your `libxml2` installation ） 中的libxml2 ，+ libxml2-devel 有的一并安装上
这是开发的源码，但有些可能检测不到的包名可以把前面error `xml2-config` not found 中 xml2-config 百度其支持的包，进行相应下载.
然后
```
make && make install
```
这步失败基本上就是上面的没有成功执行引起，仔细检测上面的`./configure`操作





### 安装 nginx-1.13.12
依然是`./configure` make make install
```
#可能遇到
./configure: error: the HTTP rewrite module requires the PCRE library.
You can either disable the module by using --without-http_rewrite_module
option, or install the PCRE library into the system, or build the PCRE library
statically from the source with nginx by using --with-pcre=<path> option.
```

pcre pcre-devel
```
# yum search pcre              
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
=================================================================================== N/S matched: pcre ====================================================================================
ghc-pcre-light-devel.x86_64 : Haskell pcre-light library development files
mingw32-pcre.noarch : MinGW Windows pcre library
mingw32-pcre-static.noarch : Static version of the mingw32-pcre library
mingw64-pcre.noarch : MinGW Windows pcre library
mingw64-pcre-static.noarch : Static version of the mingw64-pcre library
pcre-devel.i686 : Development files for pcre
pcre-devel.x86_64 : Development files for pcre
pcre-static.i686 : Static library for pcre
pcre-static.x86_64 : Static library for pcre
pcre-tools.x86_64 : Auxiliary utilities for pcre
pcre2-devel.i686 : Development files for pcre2
pcre2-devel.x86_64 : Development files for pcre2
pcre2-static.i686 : Static library for pcre2
pcre2-static.x86_64 : Static library for pcre2
pcre2-tools.x86_64 : Auxiliary utilities for pcre2
pcre2-utf16.i686 : UTF-16 variant of PCRE2
pcre2-utf16.x86_64 : UTF-16 variant of PCRE2
pcre2-utf32.i686 : UTF-32 variant of PCRE2
pcre2-utf32.x86_64 : UTF-32 variant of PCRE2
ghc-pcre-light.x86_64 : Perl5 compatible regular expression library
opensips-regex.x86_64 : RegExp via PCRE library
pcre.i686 : Perl-compatible regular expression library
pcre.x86_64 : Perl-compatible regular expression library
pcre2.i686 : Perl-compatible regular expression library
pcre2.x86_64 : Perl-compatible regular expression library

  Name and summary matches only, use "search all" for everything.
# yum install pcre.x86_64  pcre-devel.x86_64
```
直接安装即可


nginx 在编译安装时能添加许多包，如 ./configure --with-debug ,就是能打开debug模式，能够在log 上输出请求这整个过程
输出由 http 到 server 到location的生命周期发生的日志，包括参数等

- with-pcre-jit   --用于pcre的动态编译，听说动态编译可以省去静态资源的选择语句的时间，安装了正则表达式会快点，./configure的时候追加
- with-ipv6 能够监听ip6 的地址，listen ip4:port => listen ip6:port https://blog.csdn.net/jmingbh/article/details/69388647
- with-http_ssl_module 能够开启ssl 支持，即https
- with-http_stub_status_module 能够开启nginx监听 https://www.cnblogs.com/94cool/p/3872492.html
- with-http_realip_module 获取真实IP 尤其是经过代理啊，负载均衡之类 的 https://blog.csdn.net/cscrazybing/article/details/50789234
- with-http_auth_request_module 认证，复习 https://www.cnblogs.com/wangxiaoqiangs/p/6184181.html
- with-http_addition_module 响应之前或者之后追加文本内容 http://www.ttlsa.com/linux/nginx-modules-ngx_http_addition_module/
- with-http_dav_module  启用ngx_http_dav_module支持（增加PUT,DELETE,MKCOL：创建集合,COPY和MOVE方法）默认情况下为关闭，需编译开启
- with-http_geoip_module 针对地区访问控制 http://ju.outofmemory.cn/entry/16264
- with-http_gunzip_module  为不支持“gzip”编码方式的客户端解压缩头部包含“Content-Encoding: gzip”响应的过滤器。https://blog.lyz810.com/article/2016/05/ngx_http_gunzip_module_doc_zh-cn/
- with-http_gzip_static_module 解压缩头部包
- with-http_image_filter_module 来裁剪图片 https://blog.csdn.net/revitalizing/article/details/52714198
- with-http_v2_module  
- with-http_sub_module 它修改网站响应内容中的字符串 http://www.ttlsa.com/linux/nginx-modules-ngx_http_sub_module/
- with-http_xslt_module 启用ngx_http_xslt_module支持 http://www.ttlsa.com/nginx/nginx-configure-descriptions/
- with-stream 负载均衡 https://blog.csdn.net/zhiyuan_2007/article/details/71238216
- with-stream_ssl_module  ssl负载均衡 
- with-mail 邮件
- with-mail_ssl_module 邮件ssl
- with-threads https://segmentfault.com/a/1190000002924458 !import


### 安装 mysql57
mysql57 可以在mysql 官网上下载，而且提供 rpm 包，可直接用rpm -ivh安装
```
wget https://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-5.7.22-1.el7.x86_64.rpm-bundle.tar
```
当然也可以自己用了浏览器下载后拷贝到linux上
```
tar -xvf mysql-5.7.22-1.el7.x86_64.rpm-bundle.tar
# rpm -ivh *.rpm
```
可能遇到的错误~
```
warning: mysql-community-client-5.7.22-1.el7.x86_64.rpm: Header V3 DSA/SHA1 Signature, key ID 5072e1f5: NOKEY
error: Failed dependencies:
        pkgconfig(openssl) is needed by mysql-community-devel-5.7.22-1.el7.x86_64
        libaio.so.1()(64bit) is needed by mysql-community-embedded-compat-5.7.22-1.el7.x86_64
        libaio.so.1(LIBAIO_0.1)(64bit) is needed by mysql-community-embedded-compat-5.7.22-1.el7.x86_64
        libaio.so.1(LIBAIO_0.4)(64bit) is needed by mysql-community-embedded-compat-5.7.22-1.el7.x86_64
        mariadb-libs is obsoleted by mysql-community-libs-5.7.22-1.el7.x86_64
        mariadb-libs is obsoleted by mysql-community-libs-compat-5.7.22-1.el7.x86_64
        libaio.so.1()(64bit) is needed by mysql-community-server-5.7.22-1.el7.x86_64
        libaio.so.1(LIBAIO_0.1)(64bit) is needed by mysql-community-server-5.7.22-1.el7.x86_64
        libaio.so.1(LIBAIO_0.4)(64bit) is needed by mysql-community-server-5.7.22-1.el7.x86_64
        libaio.so.1()(64bit) is needed by mysql-community-server-minimal-5.7.22-1.el7.x86_64
        libaio.so.1(LIBAIO_0.1)(64bit) is needed by mysql-community-server-minimal-5.7.22-1.el7.x86_64
        libaio.so.1(LIBAIO_0.4)(64bit) is needed by mysql-community-server-minimal-5.7.22-1.el7.x86_64
        perl(Data::Dumper) is needed by mysql-community-test-5.7.22-1.el7.x86_64
        perl(JSON) is needed by mysql-community-test-5.7.22-1.el7.x86_64
```
- `pkgconfig(openssl) is needed by mysql-community-devel-5.7.22-1.el7.x86_64` 缺失 openssl
- `libaio.so.1()(64bit) is needed by mysql-community-embedded-compat-5.7.22-1.el7.x86_64` 缺失 libaio
- `perl(Data::Dumper) is needed by mysql-community-test-5.7.22-1.el7.x86_64` 缺失 perl Data 组件
- `perl(JSON) is needed by mysql-community-test-5.7.22-1.el7.x86_64` 缺失 perl JSON 组件
- `mariadb-libs is obsoleted by mysql-community-libs-compat-5.7.22-1.el7.x86_64`  mariadb-libs 已经过时
- ``ps ： 上面这句，不是叫你更换更高的版本，而是说这东东阻地方，叫你卸载它``

```
yum remove mariadb* # 卸载 mariadb
```

yum 安装 openssl libaio perl perl-JSON  autoconf（`autoconf`包含 Data::Dumper，为啥？百度的）
```
yum install openssl-devel.x86_64 openssl.x86_64 -y 
yum install yum install libaio.x86_64 libaio-devel.x86_64 -y 
yum install perl.x86_64 perl-devel.x86_64 -y 
yum install  perl-JSON.noarch -y 
yum  install autoconf -y  //此包安装时会安装perl Data:Dumper模块 
```

之后再执行
```
# rpm -ivh *.rpm
warning: mysql-community-client-5.7.22-1.el7.x86_64.rpm: Header V3 DSA/SHA1 Signature, key ID 5072e1f5: NOKEY
Preparing...                          ################################# [100%]
        file /etc/my.cnf conflicts between attempted installs of mysql-community-server-minimal-5.7.22-1.el7.x86_64 and mysql-community-server-5.7.22-1.el7.x86_64
        file /usr/bin/my_print_defaults conflicts between attempted installs of mysql-community-server-minimal-5.7.22-1.el7.x86_64 and mysql-community-server-5.7.22-1.el7.x86_64
        file /usr/bin/mysql conflicts between attempted installs of mysql-community-server-minimal-5.7.22-1.el7.x86_64 and mysql-community-client-5.7.22-1.el7.x86_64
        ...
        ...
```
还是失败 ，开头报了个 `warning: mysql-community-client-5.7.22-1.el7.x86_64.rpm: Header V3 DSA/SHA1 Signature, key ID 5072e1f5: NOKEY`
原因：这是由于yum安装了旧版本的GPG keys造成的 解决办法：后面加上  --force --nodeps 如：  `rpm -ivh *** --force --nodeps `从 RPM 版本 4.1 开始，在安装或升级软件包时会检查软件包的签名。

加上   --force --nodeps
```
rpm -ivh *.rpm --force --nodeps
```
参考资料 https://www.cnblogs.com/royfans/p/7243641.html

这次安装成功了
```
Preparing...                          ################################# [100%]
Updating / installing...
   1:mysql-community-common-5.7.22-1.e################################# [ 10%]
   2:mysql-community-libs-5.7.22-1.el7################################# [ 20%]
   3:mysql-community-client-5.7.22-1.e################################# [ 30%]
   4:mysql-community-server-5.7.22-1.e################################# [ 40%]
   5:mysql-community-test-5.7.22-1.el7################################# [ 50%]
   6:mysql-community-devel-5.7.22-1.el################################# [ 60%]
   7:mysql-community-libs-compat-8.0.1################################# [ 70%]
   8:mysql-community-embedded-compat-8################################# [ 80%]
   9:mysql-community-server-minimal-8.################################# [ 90%]
  10:mysql-community-minimal-debuginfo################################# [100%]
```

```
# service mysqld start #启动mysql 服务

# grep 'temporary password' /var/log/mysqld.log #查看mysql初始密码
2018-05-02T10:09:21.135245Z 5 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: (jmInl3q9_K5 # '(jmInl3q9_K5'  就是我这里mysql的初始密码
[root@iZbp14ame76457gp9cltecZ ~]# mysql -u root -p'(jmInl3q9_K5'
  ...
  mysql>ALTER USER 'root'@'localhost' IDENTIFIED BY 'jimb55'; # 更改密码
  ERROR 1819 (HY000): Your password does not satisfy the current policy requirements # 这句意思是你的密码不符合当前的策略要求，就是太简单
  mysql>ALTER USER 'root'@'localhost' IDENTIFIED BY '_Jimb55!'; # 换个更复习的密码
  Query OK, 0 rows affected (0.07 sec)
  mysql>
```
mysql 安装完毕!


### 可能遇到的问题
#### 1.php-fpm 安装后找不到配置问题 
```
# php-fpm -t
[03-May-2018 09:49:52] ERROR: failed to open configuration file '/usr/local/es/php7/etc/php-fpm.conf': No such file or directory (2)
[03-May-2018 09:49:52] ERROR: failed to load configuration file '/usr/local/es/php7/etc/php-fpm.conf'
[03-May-2018 09:49:52] ERROR: FPM initialization failed
```
说明没有找到PHP-fpm 配置文件 ，到目录下一看
```
# pwd
/usr/local/es/php7/etc
# ls -al
total 168
drwxr-xr-x 3 root root  4096 May  3 09:48 .
drwxr-xr-x 9 root root  4096 Apr 25 16:42 ..
-rw-r--r-- 1 root root  1277 Apr 25 16:42 php-fpm.conf.default  # 配置文件是有的，但你得把他改名字才能使用
drwxr-xr-x 1 root root  1277 Apr 25 16:42 php-fpm.d
# mv php-fpm.conf.default php-fpm.conf
```

#### 2.php 安装后找不到 php.ini 问题 
```
# php --ini
Configuration File (php.ini) Path: /usr/local/es/php7/etc/   #ini 路径
Loaded Configuration File:         (none)  # 没有可用配置文件
Scan for additional .ini files in: (none)
Additional .ini files parsed:      (none)
```
php 安装后的ini文件一般不会到安装目录下，如果需要得从解压后的文件夹上拷贝
就是之前 解压 `php-7.2.3.tar.gz` 的目录
```
# cd  ~/local/php-7.2.3 -al  # 解压php源码包后的目录
# ls  -al 
# ls  -al  | grep php.ini
-rw-rw-r--  1 root root   70218 Feb 28 00:32 php.ini-development # 开发环境版
-rw-rw-r--  1 root root   70490 Feb 28 00:33 php.ini-production  # 生产环境版
```
开发环境版 和 生产环境版 就个名字，配置有些地方不同而已，随便一份拷贝到 `Configuration File (php.ini) Path: /usr/local/es/php7/etc/`
改名为 php.ini 便可以使用！

#### 3.php扩展安装
php 有部分扩展可以在编译安装时 ./Configura 时选上，但有些开发的扩展或一些源码包上的没有的扩展就不能了

到 php.ini 中打开 pdo 扩展打开
```
extension=pdo_mysql #把;去掉
```

查看php 模块是否正常打开
```
# php -m
PHP Warning:  PHP Startup: Unable to load dynamic library 'pdo_mysql' (tried: /usr/local/es/php7/lib/php/extensions/no-d
ebug-non-zts-20170718/pdo_mysql (/usr/local/es/php7/lib/php/extensions/no-debug-non-zts-20170718/pdo_mysql: cannot open 
shared object file: No such file or directory), /usr/local/es/php7/lib/php/extensions/no-debug-non-zts-20170718/pdo_mysql.so 
(/usr/local/es/php7/lib/php/extensions/no-debug-non-zts-20170718/pdo_mysql.so: cannot open shared object file: No such file 
or directory)) in Unknown on line 0
```
明显不存在扩展包

方法一： 编译安装
到官网上下载扩展源码包（当然，解压包上就有一堆扩展包可用）
> 至于其他的开发包 ，可到 http://pecl.php.net 搜索


到php7 tar 解压目录下找 pdo_mysql
```
# cd /root/local/php-7.2.3/ext/pdo_mysql #ext包含一些基础拓展包
```
使用 phpize 生成 当前源码包的 `./configure` 文件
> ps: phpize 在 php 安装目录下的 bin 文件夹，例如我的就是 /usr/local/es/php7/bin/phpize 

```
# /usr/local/es/php7/bin/phpize 
Configuring for:
PHP Api Version:         20170718
Zend Module Api No:      20170718
Zend Extension Api No:   320170718

./configure --with-php-config=/usr/local/es/php7/bin/php-config --with-pdo-mysql=/usr/
 ...
```

> ps 这里 有个坑，--with-php-config 对应php目录下bin 有，坑在 --with-pdo-mysql上，这里需求指定mysql 的安装目录
> 但是我这里的mysql8 是按照 rpm 包安装的，尽管find / -name mysql的所有路径都试过了，但还是不行，最后找了找指定 /usr/ 
> 然后就成功了！我也不懂，大概是rpm 的问题吧，如果mysql 也是用编译安装的 --with-pdo-mysql 指定 安装路径即可
> 当然还有种做法是 指到/usr/bin/mysql_config（看个人情况，找出 mysql_config 的位置，装完mysql都会自带的）

`./configure --with-php-config=/usr/local/es/php7/bin/php-config --with-pdo-mysql=/usr/bin/mysql_config`

如果不行，可能是头文件的问题，可
`ln -s /usr/local/mysql/include/* /usr/local/include/` 
``路径根据个人情况 参考`` 
这是因为在编译时需要mysql的头的文件,而它按默认搜索找不到头文件的位置,所以才出现这个问题.
所以要将 /usr/local/mysql/include/ 目录下的mysql头文件链接到 /usr/local/include/ 的目录下.

`make` , `make install`

安装完成后，尝试以下代码进行测试
```php
<?php

    $PDO = new PDO('mysql:host=127.0.0.1;dbname=mysql', 'root', '_Jimb55!');
    $result = $PDO->query('select * from user');
    $row = $result->fetch(PDO::FETCH_ASSOC);
    print_r($row);

    // 关闭mysql连接
    $PDO = null;
```
然后报了个错
```
Fatal error: Uncaught PDOException: SQLSTATE[HY000] [2005] Unknown MySQL server host '127.0.0.1' 
(0) in /usr/local/es/nginx/html/pdo.php:3 Stack trace: 
#0 /usr/local/es/nginx/html/pdo.php(3): PDO->__construct('mysql:host=127....', 'root', '_Jimb55!') 
#1 {main} thrown in /usr/local/es/nginx/html/pdo.php on line 3
```
这是当前127.0.0.1 不允许操作数据库问题
到mysql终端执行操作
```
mysql> use mysql;
mysql> update user set host='127.0.0.1' where user='root' and host='localhost';
mysql> FLUSH PRIVILEGES; # 刷新
```
然后再次链接就成功了
> ps : mysql 远程链接设置  https://blog.csdn.net/jian1jian_/article/details/53812130



方法二： pecl 安装
若果想安装一些php7 解压目录ext里 ，不存在的包，可以用 pecl 安装，实质pecl 会到 http://pecl.php.net 上搜索下载，并安装
当然，自己也可以到http://pecl.php.net上下载包，进行编译安装也行（方法一）

`pecl 位于php 安装目录的bin 文件夹下`

```
[root@iZbp14ame76457gp9cltecZ bin]# pwd
/usr/local/es/php7/bin
[root@iZbp14ame76457gp9cltecZ bin]# ./pecl search swoole
WARNING: channel "pecl.php.net" has updated its protocols, use "pecl channel-update pecl.php.net" to update
Retrieving data...0%
.Matched packages, channel pecl.php.net:
=======================================
Package          Stable/(Latest) Local
swoole           2.1.3 (stable)  2.1.3 Event-driven asynchronous and concurrent networking engine with high performance for PHP.
swoole_serialize 0.1.1 (beta)          the fastest and smallest serialize fucntion bound for php7
```

> ps : 上面有一个 **WARNING: channel "pecl.php.net" has updated its protocols, use "pecl channel-update pecl.php.net" to update** 的错误
> 这个不用理会，如果真的执行了pecl channel-update pecl.php.net ，在你安装包时反倒会报一个**Connection to `ssl://pecl.php.net:443' failed: Unable to find the socket transport "ssl" - did you forget to enable it when you configured PHP?** 错误
> 不必理会就行

安装swoole！
```
# ./pecl install swoole
...
Build process completed successfully
Installing '/usr/local/es/php7/lib/php/extensions/no-debug-non-zts-20170718/swoole.so'
install ok: channel://pecl.php.net/swoole-2.1.3
configuration option "php_ini" is not set to php.ini location
You should add "extension=swoole.so" to php.ini
```
最后如上所说 `You should add "extension=swoole.so" to php.ini` 把  `extension=swoole.so` 添加到  `php.ini`
再 `php -m` 确认一下 包是否安装上！ `php-fpm` 记得重启一下!
