---
title: 'mysql 编译安装5.7'
date: 2018-05-14
tags:
  - linux
  - mysql
toc: true
toc_label: "目录"
header:
  teaser: /assets/images/blogs/mysql-install/mysql-install.png
---

## mysql 编译安装
mysql 源码包下载

> 链接 https://dev.mysql.com/downloads/mysql/5.7.html#downloads


![Image text](/assets/images/blogs/mysql-install/mysqlyuanmbdow.png)

```
[root@localhost mysql-5.7.22]# tar -zxvf mysql-5.7.22.tar.gz
[root@localhost mysql-5.7.22]# cd mysql-5.7.22
[root@localhost mysql-5.7.22]# yum install -y gcc c ncurses-devel cmake libaio bison gcc-c++  git (mysql需要的环境与工具)
```
mysql5.7 需求环境与工具 https://dev.mysql.com/doc/refman/5.7/en/source-installation.html

```
[root@localhost mysql-5.7.22]#  cmake  .  -DCMAKE_INSTALL_PREFIX=/usr/local/es/mysql57/ \
-DMYSQL_UNIX_ADDR=/tmp/mysql.sock \
-DMYSQL_DATADIR=/data/mysql \
-DSYSCONFDIR=/etc \
-DMYSQL_USER=mysql \
-DMYSQL_TCP_PORT=3306 \
-DWITH_XTRADB_STORAGE_ENGINE=1 \
-DWITH_INNOBASE_STORAGE_ENGINE=1 \
-DWITH_PARTITION_STORAGE_ENGINE=1 \
-DWITH_BLACKHOLE_STORAGE_ENGINE=1 \
-DWITH_MYISAM_STORAGE_ENGINE=1 \
-DWITH_READLINE=1 \
-DENABLED_LOCAL_INFILE=1 \
-DWITH_EXTRA_CHARSETS=1 \
-DDEFAULT_CHARSET=utf8 \
-DDEFAULT_COLLATION=utf8_general_ci \
-DEXTRA_CHARSETS=all \
-DWITH_BIG_TABLES=1 \
-DWITH_DEBUG=0
```

**官方 工具选项解析**
- -DCMAKE_INSTALL_PREFIX=/usr/local/es/mysql57/ \   #安装目录
- -DMYSQL_UNIX_ADDR=/tmp/mysql.sock \   服务器侦听套接字连接的Unix套接字文件路径，默认/tmp/mysql.sock。 这个值可以在服务器启动时用--socket选项来设置。所以这条可以去掉
- -DMYSQL_DATADIR=/data/mysql \  MySQL数据目录的位置。 该值可以在服务器启动时使用--datadir选项进行设置。
- -DSYSCONFDIR=/etc \  默认的my.cnf选项文件目录。 此位置不能在服务器启动时设置，但可以使用--defaults-file = file_name选项使用给定的选项文件启动服务器，其中file_name是该文件的完整路径名。
- -DMYSQL_USER=mysql \     指定用户名
- -DMYSQL_TCP_PORT=3306 \  服务器侦听TCP / IP连接的端口号。默认值是3306。 该值可以在服务器启动时使用--port选项进行设置。
- -DWITH_XTRADB_STORAGE_ENGINE=1 \   储存引擎 XTRADB
- -DWITH_INNOBASE_STORAGE_ENGINE=1 \ 储存引擎 INNOBASE
- -DWITH_PARTITION_STORAGE_ENGINE=1 \储存引擎 PARTITION
- -DWITH_BLACKHOLE_STORAGE_ENGINE=1 \储存引擎 BLACKHOLE
- -DWITH_MYISAM_STORAGE_ENGINE=1 \   储存引擎 MYISAM
- -DWITH_READLINE=1 \
- -DENABLED_LOCAL_INFILE=1 \  该选项控制MySQL客户端库的已编译默认LOCAL功能???啥意思
- -DWITH_EXTRA_CHARSETS=1 \   这个为什么是1，文档不是name ，字符串吗？  要包含哪些额外的字符集： all complex none
- -DDEFAULT_CHARSET=utf8 \   服务器字符集。默认情况下，MySQL使用latin1（cp1252西欧）字符集。  该值可以在服务器启动时使用--character_set_server选项进行设置。
- -DDEFAULT_COLLATION=utf8_general_ci \  服务器整理。默认情况下，MySQL使用latin1_swedish_ci。该值可以在服务器启动时使用--character_set_server选项进行设置。
- -DEXTRA_CHARSETS=all \   
- -DWITH_BIG_TABLES=1 \
- -DWITH_DEBUG=0  是否包含调试支持。

>ps : 储存引擎可用  -DWITH_{engine}_STORAGE_ENGINE=1 方式打开,如上 {engine} 可选

mysql 5.7 cmake 配置 https://dev.mysql.com/doc/refman/5.7/en/source-configuration-options.html#option_cmake_with_boost
mysql 5.7 字符集 查看 https://dev.mysql.com/doc/refman/5.7/en/show-collation.html


报了个错

```
CMake Error at cmake/boost.cmake:81 (MESSAGE):
  You can download it with -DDOWNLOAD_BOOST=1 -DWITH_BOOST=<directory>

  This CMake script will look for boost in <directory>.  If it is not there,
  it will download and unpack it (in that directory) for you.

  If you are inside a firewall, you may need to use an http proxy:

  export http_proxy=http://example.com:80

Call Stack (most recent call first):
  cmake/boost.cmake:238 (COULD_NOT_FIND_BOOST)
  CMakeLists.txt:506 (INCLUDE)
```

原因是 根据官方 mysql5.7 需求环境与工具 中

The Boost C++ libraries are required to build MySQL (but not to use it). Boost 1.59.0 must be installed. To obtain Boost and its installation instructions, visit the official site. After Boost is installed, tell the build system where the Boost files are located by defining the WITH_BOOST option when you invoke CMake. For example:
```
shell> cmake . -DWITH_BOOST=/usr/local/boost_1_59_0
```
Adjust the path as necessary to match your installation.(copy in mysql.com)

需要 Boost库
>ps ：Boost库是一个可移植、提供源代码的C++库，作为标准库的后备，是C++标准化进程的开发引擎之一。

解决办法 ： 在上面的预编译命令上加上 
```
-DDOWNLOAD_BOOST=1 \
-DWITH_BOOST=/root/local/mysql-5.7.22/boost  #boost在mysql 解压目录
```

> ps : 注意上面的下载截图上下载的 mysql 5.7.22 有两个版本  普通版本和Includes Boost Headers版本 ,Boost 是 Includes Boost Headers才带的

```
[root@localhost mysql57]# make #接着是make
[root@localhost mysql57]# make install
```

> ps: 我make install这里还报了个错,原因是系统硬盘容量不足（虚拟机的）,上面是系统扩容后执行成功的过程

```
[root@localhost mysql57]# cd /usr/local/es/mysql57
[root@localhost mysql57]# pwd
/usr/local/es/mysql57

[root@localhost mysql57]# cp support-files/mysql.server /etc/init.d/mysqld  #为了能在 service 上启动mysqld 服务
[root@localhost mysql57]# chkconfig --add mysqld  # systemctl 里的服务 注册
[root@localhost mysql57]# chkconfig --level 35 mysqld on # 打开

[root@localhost mysql57]# mkdir mysql-files
[root@localhost mysql57]# chown mysql:mysql mysql-files -R #确保 mysql 用户和mysql用户组已经创建 ,没有的要自行创建
[root@localhost mysql57]# chmod 750 mysql-files -R
```

修改my.cnf 配置文件

```
[root@localhost mysql57]# vi /etc/my.cnf
[mysqld]
# 修改到需要的目录，前提是mysql用户和组有权限，mysql-files是上面创建的文件夹，已赋予mysql用户权限
basedir = /usr/local/es/mysql57
datadir = /data/mysql
socket  = /tmp/mysql.sock
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
# Settings user and group are ignored when systemd is used.
# If you need to run mysqld under a different user or group,
# customize your systemd unit file for mariadb according to the
# instructions in http://fedoraproject.org/wiki/Systemd

[mysqld_safe]
# mysqld_safe 启动报错修改这两个
# log-error=/usr/local/es/mysql57/mysql-files/logs/mariadb.log 
# pid-file=/usr/local/es/mysql57/mysql-files/logs/mariadb.pid

#
# include all files from the config directory
#
# !includedir /etc/my.cnf.d
```

初始化mysql

```
[root@localhost mysql57]# groupadd mysql
[root@localhost mysql57]# useradd -g mysql  -r mysql
[root@localhost mysql57]# bin/mysqld  --initialize --user=mysql --datadir=/data/mysql  --basedir=/usr/local/es/mysql57
2018-05-15T12:03:38.914370Z 0 [Warning] TIMESTAMP with implicit DEFAULT value is deprecated. Please use --explicit_defaults_for_timestamp server option (see documentation for more details).
2018-05-15T12:03:39.099197Z 0 [Warning] InnoDB: New log files created, LSN=45790
2018-05-15T12:03:39.178042Z 0 [Warning] InnoDB: Creating foreign key constraint system tables.
2018-05-15T12:03:39.245722Z 0 [Warning] No existing UUID has been found, so we assume that this is the first time that this server has been started. Generating a new UUID: 00f7dd1a-5838-11e8-a364-000c29920633.
2018-05-15T12:03:39.247497Z 0 [Warning] Gtid table is not ready to be used. Table 'mysql.gtid_executed' cannot be opened.
2018-05-15T12:03:39.248853Z 1 [Note] A temporary password is generated for root@localhost: tJ!i/kOlu487  #生成的临时密码,记住
```

执行成功！准备启动mysqld服务

```
[root@localhost mysql57]# service mysqld start
Starting MySQL.Logging to '/data/mysql/localhost.localdomain.err'.
 SUCCESS! 
```

添加两个命令的软连接

```
[root@localhost mysql57]#ln  -s  /usr/local/es/mysql57/bin/mysql /usr/bin/ 
[root@localhost mysql57]#ln  -s  /usr/local/es/mysql57/bin/mysqld /usr/bin/
[root@localhost mysql57]#mysql -uroot -p'tJ!i/kOlu487' 
mysql>ALTER USER 'root'@'localhost' IDENTIFIED BY '_Jimb55!'; # 换个更复习的密码
Query OK, 0 rows affected (0.07 sec)
mysql> exit;
Bye
[root@localhost mysql57]# mysql -uroot -p'_Jimb55'  
mysql> ... 
```

操作完成！

## 可能出现的错误
1 输入上面的密码时提示 `mysql: [Warning] Using a password on the command line interface can be insecure.`
```
# mysql -uroot -p
Enter password：
```
转换成如此输入即可

2 mysql client 登录时 报`ERROR 2002 (HY000): Can't connect to local MySQL server through socket '/tmp/mysql.sock' (2)`

```
# find / -name mysql.sock # 看看有没有输出和路径对上了没有，没输出表示mysqld.service根本没有启动成功
/var/lib/mysql/mysql.sock # 如我此处的路径就跟报错提示中的 /tmp/mysql.sock 不同，所以报错
```

只能修改配置文件

```
[client]
port = 3306 
socket = /var/lib/mysql/mysql.sock #此处修改为你find 出来的mysql.sock路径

[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
basedir = /usr/local/es/mysql57

# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
# Settings user and group are ignored when systemd is used.
# If you need to run mysqld under a different user or group,
# customize your systemd unit file for mariadb according to the
# instructions in http://fedoraproject.org/wiki/Systemd

[mysqld_safe]
#log-error=/var/log/mariadb/mariadb.log
#pid-file=/var/run/mariadb/mariadb.pid

#
# include all files from the config directory
#
#!includedir /etc/my.cnf.d

```
再次链接应该没问题了


## mysql55 编译安装
```text
#wget http://down1.chinaunix.net/distfiles/mysql-5.5.20.tar.gz
#tar -zxvf mysql-5.5.20.tar.gz
#cd mysql-5.5.20
#yum install -y gcc c ncurses-devel cmake libaio bison gcc-c++  git   ncurses 
#cmake  .  -DCMAKE_INSTALL_PREFIX=/usr/local/mysql55/ \
-DMYSQL_UNIX_ADDR=/tmp/mysql.sock \
-DMYSQL_DATADIR=/data/mysql \
-DSYSCONFDIR=/etc \
-DMYSQL_USER=mysql \
-DMYSQL_TCP_PORT=3306 \
-DWITH_INNOBASE_STORAGE_ENGINE=1 \
-DWITH_MYISAM_STORAGE_ENGINE=1 \
-DWITH_READLINE=1 \
-DENABLED_LOCAL_INFILE=1 \
-DWITH_EXTRA_CHARSETS=1 \
-DDEFAULT_CHARSET=utf8 \
-DDEFAULT_COLLATION=utf8_general_ci \
-DEXTRA_CHARSETS=all \
-DWITH_BIG_TABLES=1 \
-DWITH_DEBUG=0
#make &&make install

#cd /usr/local/mysql55/ 
#cp  -f  support-files/my-large.cnf /etc/my.cnf
#cp  -f support-files/mysql.server /etc/init.d/mysqld 
#chkconfig --add mysqld 
#chkconfig --level 35 mysqld on
#mkdir -p  /data/mysql
#useradd  mysql
#/usr/local/mysql55/scripts/mysql_install_db  --user=mysql --datadir=/data/mysql/ --basedir=/usr/local/mysql55/ 
#ln  -s  /usr/local/mysql55/bin/mysql /usr/bin/mysql
#service  mysqld  restart
#mysql -uroot -p'new-password'
mysql> update mysql.user set password=password("_Jimb55!") where user="root";
mysql> flush privileges;
```