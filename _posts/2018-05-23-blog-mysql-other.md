---
title: 'mysql 知识汇总'
date: 2018-05-23
tags:
  - linux
  - mysql
toc: true
toc_label: "目录"
header:
  teaser: /assets/images/blogs/mysql-install/mysql-install.png
---

## 忘记密码

```
root@ubuntu:~# mysqld_safe  --user=mysql --skip-grant-tables &  
2018-05-23T10:17:30.783228Z mysqld_safe Logging to syslog.
2018-05-23T10:17:30.786454Z mysqld_safe Logging to '/var/log/mysql/error.log'.
2018-05-23T10:17:30.802883Z mysqld_safe Starting mysqld daemon with databases from /var/lib/mysql
```

若是提示 `UNIX socket file don't exists.`， 
```
root@ubuntu:~# mysqld_safe  --user=mysql --skip-grant-tables &         
[1] 32385
root@ubuntu:~# 2018-05-23T10:08:08.348893Z mysqld_safe Logging to syslog.
2018-05-23T10:08:08.351959Z mysqld_safe Logging to '/var/log/mysql/error.log'.
2018-05-23T10:08:08.366142Z mysqld_safe Directory '/var/run/mysqld' for UNIX socket file don't exists.

# 执行以下
root@ubuntu:~# mkdir -p /var/run/mysqld
root@ubuntu:~# chown mysql:mysql /var/run/mysqld
```


进入mysql

```
root@ubuntu:~# mysql
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 3
Server version: 5.7.19-0ubuntu0.16.04.1-log (Ubuntu)

Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| ecshop             |
| mysql              |
| ops                |
| performance_schema |
| sys                |
| test               |
+--------------------+
7 rows in set (0.04 sec)

mysql> clear
mysql> use mysql;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
#修改密码
mysql> update user set authentication_string=password('_Jimb55!') where user='root';               
Query OK, 1 row affected, 1 warning (0.03 sec)
Rows matched: 1  Changed: 1  Warnings: 1
```
> ps 注意实验使用的mysql 版本是 5.7，<5.7 的版本 把 authentication_string 改为 password;

修改密码成功后 kill 掉safe mysql 程序
```
root@ubuntu:~# ps -ef | grep mysql
root      36174  33211  0 03:17 pts/0    00:00:00 /bin/sh /usr/bin/mysqld_safe --user=mysql --skip-grant-tables
root      37102  33211  0 03:30 pts/0    00:00:00 grep --color=auto mysql
root@ubuntu:~# kill -9 36174
root@ubuntu:~# service mysql stop
root@ubuntu:~# ps -ef | grep mysql
root      37332  33211  0 03:32 pts/0    00:00:00 grep --color=auto mysql
root@ubuntu:~# service mysql start
root@ubuntu:~# mysql
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: NO)
root@ubuntu:~# mysql -uroot -p'_Jimb55!'
mysql> 
```


## 配置缓慢sql 日志
就是记录那些sql过慢用的

```text
mysql> show variables like  "%slow%";
+---------------------------+--------------------------------+
| Variable_name             | Value                          |
+---------------------------+--------------------------------+
| log_slow_admin_statements | OFF                            |
| log_slow_slave_statements | OFF                            |
| slow_launch_time          | 2                              |
| slow_query_log            | ON                             |
| slow_query_log_file       | /var/lib/mysql/ubuntu-slow.log |
+---------------------------+--------------------------------+
5 rows in set (0.00 sec)

mysql> show variables like  "%long_query%";
+-----------------+----------+
| Variable_name   | Value    |
+-----------------+----------+
| long_query_time | 0.010000 |
+-----------------+----------+
1 row in set (0.00 sec)

mysql> 
```

slow_query_log          是否开启慢查询日志
long_query_time		    慢查询超时时间，默认为10s，MYSQL5.5以上可以设置微秒；
slow_query_log_file 	慢查询日志文件；
slow_launch_time        Thread create时间，单位秒，如果thread create的时间超过了这个值，该变量slow_launch_time的值会加1；
log-queries-not-using-indexes  记录未添加索引的SQL语句。

开启
```text
###当前实验使用的版本5.7 配置
slow_query_log = on
#slow-query-log-file = /data/mysql/localhost.log
long_query_time = 0.01
log-queries-not-using-indexes

### 5.6 之前
log-slow-queries = /data/mysql/localhost.log 
long_query_time = 0.01
log-queries-not-using-indexes
```

重启 mysql 服务
```text
service mysql restart
```

当有SQL 语句 超过 0.01 ms 是，便会记录到 /var/lib/mysql/ubuntu-slow.log
```text
root@ubuntu:/# cat /var/lib/mysql/ubuntu-slow.log
/usr/sbin/mysqld, Version: 5.7.19-0ubuntu0.16.04.1-log ((Ubuntu)). started with:
Tcp port: 3306  Unix socket: /var/run/mysqld/mysqld.sock
Time                 Id Command    Argument
# Time: 2018-05-23T13:17:39.618627Z
# User@Host: root[root] @ localhost []  Id:     4
# Query_time: 0.026244  Lock_time: 0.000260 Rows_sent: 1  Rows_examined: 36985
use test;
SET timestamp=1527081459;
select count(*),sum(time) from pierce left join gek_user as p on pierce.id = p.id  where pierce.name like '%bxxx2%';
# Time: 2018-05-23T13:18:16.201251Z
# User@Host: root[root] @ localhost [127.0.0.1]  Id:     5
# Query_time: 0.110119  Lock_time: 0.000184 Rows_sent: 1  Rows_examined: 73952
SET timestamp=1527081496;
select count(*),sum(time) from pierce left join gek_user as p on pierce.id = p.id  where pierce.name like '%b%';
```
> ps 注意信息中的ID 表示同一链接所执行的sql 
> 还有`Lock_time`，`mysql` 慢查询日志不能记录 `lock_time` 过长，但实际sql 执行时间没有达到`long_query_time`的记录

参考资料:[详解MySQL慢日志（上）query_time\start_time\lock_time 的坑](http://blog.itpub.net/29773961/viewspace-2147315/)

可使用 mysqldumpslow 来分析查询slow的列表
```text
root@ubuntu:/mnt/hgfs/VMworkspace/github.io# mysqldumpslow -s t -t 2          

Reading mysql slow query log from /var/lib/mysql/ubuntu-slow.log
Count: 1  Time=313.06s (313s)  Lock=0.00s (0s)  Rows=0.0 (0), root[root]@localhost
  insert into pierce(`name`,`key`,`time`) values('S','S','S')

Count: 3  Time=2.33s (7s)  Lock=0.00s (0s)  Rows=1.0 (3), root[root]@localhost
  select sleep(N),N
```


> ps 注意我的日志是 /var/lib/mysql/ubuntu-slow.log ，不是/data/mysql/localhost.log ，配置/data/mysql/localhost.log 可能出现权限问题，如下

```text
root@ubuntu:/# mkdir /data/mysql -p
root@ubuntu:/# touch /data/mysql/localhost.log
root@ubuntu:/# chmod 777 /data/
root@ubuntu:/# chown -R mysql:mysql /data/mysql
```
查看错误日志 ，路径根据实际情况而定

```text
root@ubuntu:/# service mysql start
root@ubuntu:/# cat /var/log/mysql/error.log
...
mysqld: File '/data/mysql/localhost.log' not found (Errcode: 13 - Permission denied)
2018-05-23T13:10:27.729600Z 0 [ERROR] Could not use /data/mysql/localhost.log for logging (error 13 - Permission denied). Turning logging off for the server process. To turn it on again: fix the cause, then either restart the query logging by using "SET GLOBAL SLOW_QUERY_LOG=ON" or restart the MySQL server.
```
明明已经给足权限却还报错，可以参考以下解决办法
参考资料:[Linux下MySQL的写文件时权限错误(Errcode: 13)解决方法](https://www.linuxidc.com/Linux/2012-02/55533.htm)

selinux关闭方式 centos
```text
shell> vi /etc/selinux/config
SELINUX=disabled
```

## Explain 分析 SQL

## profiles 查询性能分析



