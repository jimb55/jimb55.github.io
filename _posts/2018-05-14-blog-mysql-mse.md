---
title: 'mysql 主从同步'
date: 2018-05-16
tags:
  - linux
  - mysql
toc: true
toc_label: "目录"
header:
  teaser: /assets/images/blogs/mysql-install/mysql-install.png
---

## 数据库主从分离原理
mysql 主从分离写入的数据需要同步，而实现数据同步的文件就是binlog日志和relaylog日志，
master 通过吧写入和更新的 sql 语句转换到二进制的binlog文件,共享（不是发送，是共享，需要来自己来拿的共享）给授权的slave端
slave 通过异步IO 线程读取和 position 点 读取需要的binlog内容和最新的position点，写入到relaylog,同时开启的异步SQL线程检测relaylog更新而转换成SQL语句进行执行
如此实现master 和 slave 的数据同步

流程如下
1. slave 通过 异步IO线程 请求 master
2. master 判断用户是否已授权
3. slave 异步IO线程 向master 发送 position 点
4. master 的 异步IO线程 根据position点 读取 binlog 内容
5. master 异步IO线程 发送 binlog 内容 和 当前binlog 最新的 position（方便下次slave在此处读取）
6. slave 异步IO线程  把position点记录到master.info 和 binlog 内容写到 relaylog 文件（末端）
7. slave 异步SQL线程检测relaylog变化，把其新内容转换为sql 语句进行执行
8. 周而复始

如下图所示
![Image text](/assets/images/blogs/mysql-mse/yl.jpg)


## 配置实验
准备两台linux且安装好mysql的机子

### master 172.16.47.134

修改配置文件，开启bin-log
```
[root@localhost mysql57]# vi /etc/my.cnf
[mysqld]
# ...
log-bin=mysql-bin
server-id       = 1
binlog_format=mixed
# ...

[root@localhost mysql57]# service mysqld restart #重启服务
[root@localhost mysql57]# mysql -uroot -p'_Jimb55!'
mysql> create database testb charset=utf8;
mysql> use testb;
Database changed
mysql> create table students(
    -> id int auto_increment primary key,
    -> sname varchar(10) not null
    -> );
Query OK, 0 rows affected (0.02 sec)

mysql> show tables;
+-----------------+
| Tables_in_testb |
+-----------------+
| students        |
+-----------------+
1 row in set (0.00 sec)
mysql> insert into students(sname) values("tang");
Query OK, 1 row affected (0.03 sec)
mysql> insert into students(sname) values("zhang");
Query OK, 1 row affected (0.03 sec)
mysql> insert into students(sname) values("wang");
Query OK, 1 row affected (0.03 sec)
mysql> select * from students;
+----+-------+
| id | sname |
+----+-------+
|  1 | tang  |
|  2 | zhang |
|  3 | wang  |
+----+-------+
3 rows in set (0.01 sec)
```

创建测试表和数据库备用，作为实验主从分离，接着是开启从服务器用户的注册
```
mysql> grant  replication  slave  on *.* to  'salveruser'@'%'  identified by  '123456';
mysql> show master status;
       +------------------+----------+--------------+------------------+-------------------+
       | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
       +------------------+----------+--------------+------------------+-------------------+
       | mysql-bin.000005 |     2117 |              |                  |                   |
       +------------------+----------+--------------+------------------+-------------------+
       1 row in set (0.00 sec)

```

然后导出数据库

```
[root@localhost mysql57]# bin/mysqldump -uroot -p'_Jimb55!' testb > testb.sql
mysqldump: [Warning] Using a password on the command line interface can be insecure.
[root@localhost mysql57]# ls -al | grep sql
drwxr-xr-x. 10 root root  4096 5月  15 23:43 mysql-test
-rw-r--r--.  1 root root  1922 5月  16 23:52 testb.sql
```


### slave 172.16.47.132

拷贝并导入数据库到本地
```
[root@localhost mysql57]# scp  root@172.16.47.134:/usr/local/es/mysql57/testb.sql /usr/local/es/mysql57/testb.sql
root@172.16.47.134's password: 
testb.sql             100%    0     0.0KB/s   00:00   

[root@localhost mysql57]# mysql -uroot -p'_Jimb55!'
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 4
Server version: 5.7.22 Source distribution

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> create database testb charset=utf8;  #创建数据库
Query OK, 1 row affected (0.00 sec)

mysql> use testb;
Database changed
mysql> source /usr/local/es/mysql57/testb.sql;  导入数据库
Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)
...

mysql> select * from students;
+----+-------+
| id | sname |
+----+-------+
|  1 | tang  |
|  2 | zhang |
|  3 | wang  |
+----+-------+
3 rows in set (0.00 sec)

mysql> change master to 
    -> master_host='172.16.47.134',
    -> master_user='salveruser',
    -> master_password='123456',
    -> master_log_file='mysql-bin.000005',
    -> master_log_pos=2117;
Query OK, 0 rows affected, 2 warnings (0.01 sec)  #这句是创建主数据库服务器，以后就问它拿数据

mysql> start slave;   
Query OK, 0 rows affected (0.01 sec)  #开启slave ,mysql 5.6 应该是 slave start

mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.16.47.134
                  Master_User: salveruser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000005
          Read_Master_Log_Pos: 2117
               Relay_Log_File: localhost-relay-bin.000002
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000005
             Slave_IO_Running: Yes   #这两个需要是 yes 才算成功
            Slave_SQL_Running: Yes   #这两个需要是 yes 才算成功
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
          ...
```
>ps : `master_log_pos=2117`; 的这个点要输入你导出数据时的 pos 点，不然偏差会导致数据的偏差
>例如，我的 `sname = wang` 这条数据未输出到students 表时的 pos 点是 1547 ，我将 `master_log_pos=1547`;执行，
>那么主从同步后的 students 表就缺少 `sname = wang`  这条数据了 (上面假设是在数据库没有导入前提下，就是 上面没有执行 `source /usr/local/es/mysql57/testb.sql;` 导入数据库)


测试一下

master 插入一条数据
```
[root@localhost mysql57]# mysql -uroot -p'_Jimb55!'                          
mysql> use testb;
Database changed
mysql> select * from students;
+----+-------+
| id | sname |
+----+-------+
|  1 | tang  |
|  2 | zhang |
|  3 | wang  |
+----+-------+
3 rows in set (0.00 sec)

mysql> insert into students(sname) values("long");
Query OK, 1 row affected (0.00 sec)

mysql> select * from students;
+----+-------+
| id | sname |
+----+-------+
|  1 | tang  |
|  2 | zhang |
|  3 | wang  |
|  4 | long  |
+----+-------+
4 rows in set (0.00 sec)
```

slave查看一下
```
mysql> select * from students;
+----+-------+
| id | sname |
+----+-------+
|  1 | tang  |
|  2 | zhang |
|  3 | wang  |
|  4 | long  |
+----+-------+
4 rows in set (0.00 sec)
```
配置完成!