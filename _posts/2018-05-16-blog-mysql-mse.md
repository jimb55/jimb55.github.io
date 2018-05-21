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

### master 服务器配置（172.16.47.134）

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


### slave 服务器配置(172.16.47.132)

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
mysql> source /usr/local/es/mysql57/testb.sql;  #导入数据库
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

## 生产环境添加一台slave
生产环境的 master 每时每刻都有新的数据写入，难以取得当前数据库对应的pos点，因为pos 点随着数据的写入而不停更新的。
可能你查看数据pos点那一时刻，数据库就插入新数据，这时即使你导出数据库也没用，因为导出的数据库与你上面查看的pos 点
对应不上！

那么直接把 pos 点设置 为master新建需要同步的数据库时对应的 pos 点呢？
这样是可行，在slave 设置 master 服务器后，会自动建设数据库和请求数据，达到主从同步。
``但问题是`` 这会给 master 的IO 线程造成性能损失，新增一台 slaver 还好，十台就得穿10次，IO压力便大的可怕，需求网络传输非常高。

正确的做法是
第一步 在master端 锁上数据库写操作并刷新脏页,然后 show master status 取得 pos 点再导出数据库
第二步 slave端导入数据库再 change master设置主从分离.    
第三步 解锁master写操作

### 脏页
脏页指的是在内存中没有写到磁盘的数据，可以通过 innodb_max_dirty_pages_pct 参数调整，默认是 90%。

### 刷新脏页和数据库锁
使用命令 `FLUSH TABLES WITH READ LOCK`。这个命令是全局读锁定，执行了命令之后所有库所有表都被锁定只读。
这句命令有前提，必须将所有的脏页都要更新到磁盘，并且所有的表不在锁状态，事务完成。
`show processlist`;可查看当前语句执行线程
那么又有新请求指令添加到内存呢？`FLUSH TABLES WITH READ LOCK` 会添加上请求锁，不允许再添加新的脏页。
参考资料 [FLUSH TABLES WITH READ LOCK](https://www.cnblogs.com/sunss/archive/2012/02/02/2335960.html) 

还有注意一点是，FLUSH TABLES WITH READ LOCK 后面可能还有线程 sql 留在内存中 waiting，
所谓的刷新脏页是 执行 FLUSH TABLES WITH READ LOCK 之前的sql线程
[Image text](/assets/images/blogs/mysql-mse/next.png)


### 配置一台slave
> 准备：一台装好mysql 的linux，一台能执行php代码的机子
快速请求的php代码
```php
<?php
$mysql_conf = array(
    'host'    => '172.16.47.134:3306',
    'db'      => 'testb',
    'db_user' => 'allperson',
    'db_pwd'  => '123456',
);
$pdo = new PDO("mysql:host=" . $mysql_conf['host'] . ";dbname=" . $mysql_conf['db'], $mysql_conf['db_user'], $mysql_conf['db_pwd']);
$pdo->exec("set names 'utf8'");
$pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);

while (1){
    try{
        for($i=0;$i<10000;$i++){
            $sql = "INSERT INTO `students` (sname) VALUES ('".substr(uniqid(),6)."')";
            $pdo->exec($sql);
            echo $i.PHP_EOL;
        }
    }catch (Exception $e){
        print_r($e);
        sleep(60);
    }
    sleep(5);
}
```


master 机:
```
mysql> select * from students;
...
| 8004 | b9e2a14 |
| 8005 | b9e5124 |
| 8006 | b9e5124 |
#### 这里在另一台机子执行php 脚本，到输出 311 时执行下面脚本（程序跳太快，我是立刻切屏执行FLUSH TABLES WITH READ LOCK）
mysql> FLUSH TABLES WITH READ LOCK;
Query OK, 0 rows affected (0.00 sec)

mysql> show processlist;
+----+-----------+-------------------+-------+---------+------+------------------------------+---------------------------------------------------+
| Id | User      | Host              | db    | Command | Time | State                        | Info                                              |
+----+-----------+-------------------+-------+---------+------+------------------------------+---------------------------------------------------+
| 15 | root      | localhost         | testb | Query   |    0 | starting                     | show processlist                                  |
| 29 | allperson | 172.16.47.1:63780 | testb | Query   |   27 | Waiting for global read lock | INSERT INTO `students` (sname) VALUES ('b9e5124') |
+----+-----------+-------------------+-------+---------+------+------------------------------+---------------------------------------------------+
2 rows in set (0.00 sec)
mysql> select * from students;
...
| 8316 | b9e2a14 |
| 8317 | b9e5124 |
| 8318 | b9e5124 |
```
接着做法就是导出数据库，同步数据库，slave 配置 mysql 机子，做法与上面一样。

> ps 导出数据库这步不能退出当前 mysql 终端，退出mysql 终端后 FLUSH TABLES WITH READ LOCK; 将会失效，应新建master终端连接操作！
![Image text](/assets/images/blogs/mysql-mse/ort.png)

等都完成之后， 在master 端执行 
```
mysql>unlock tables;
```
完成操作

### 可能遇到的问题
使用比导入数据库对应pos 点要少得 pos 点
```
mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.16.47.134
                  Master_User: salveruser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000005
          Read_Master_Log_Pos: 2781294
               Relay_Log_File: localhost-relay-bin.000002
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000005
             Slave_IO_Running: Yes
            Slave_SQL_Running: No
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 1062
                   Last_Error: Error 'Duplicate entry '4' for key 'PRIMARY'' on query. Default database: 'testb'. Query: 'insert into students(sname) values("long")' #失败
```
所以得找对数据库和数据库对应的pos 点