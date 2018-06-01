---
title: 'mysql Replication 数据备份'
date: 2018-05-29
tags:
  - linux
  - mysql
toc: true
toc_label: "目录"
header:
  teaser: /assets/images/blogs/mysql-install/mysql-install.png
---
主从复制的数据备份与单台服务器不同，因为主库与从库的日志不同，需要考虑对应的**pos**点和**binlog**，当然还有对应的中继日志;
当然 在从库备份的好处是减轻 主库压力，毕竟主库备份会导致IO影响;

## mysqldump 备份

### 操作
第一步应先停止服务,这是防止 **slave** 在备份期间受到主库发送过来的信息影响
```text
[root@localhost ~]# mysql -uroot -p'_Jimb55!'
mysql> stop slave SQL_THREAD;
Query OK, 0 rows affected (0.01 sec)
mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.16.47.134
                  Master_User: salveruser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000014
          Read_Master_Log_Pos: 36238
               Relay_Log_File: localhost-relay-bin.000032
                Relay_Log_Pos: 36451
        Relay_Master_Log_File: mysql-bin.000014
             Slave_IO_Running: Yes
            Slave_SQL_Running: No
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
```
>ps 为啥只停止 SQL_THREAD呢，因为sql 线程是读取中继日志而更新数据库的，影响也是它造成的，但是IO 线程时传输数据的，
> 只会更新 中继日志，却不会更新 数据库。当然你直接执行`stop slave`也是可以的，就怕备份时不更新中继日志，到了备份完更新造成网络IO频繁问题而已。

第二步 是导出 数据库
```text
[root@localhost bf]# mysqldump --all-databases > all.dump -uroot -p'_Jimb55!'
[root@localhost bf]# ls -al
总用量 1044
drwxr-xr-x. 2 root root      22 5月  29 16:40 .
drwxr-xr-x. 4 root root      69 5月  29 16:40 ..
-rw-r--r--. 1 root root 1067924 5月  29 16:40 all.dump
```

第三步是在第二步成功的前提下启动slaver
```text
[root@localhost bf]# mysql -e 'start slave;show slave status \G;' -uroot -p'_Jimb55!'
mysql: [Warning] Using a password on the command line interface can be insecure.
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.16.47.134
                  Master_User: salveruser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000014
          Read_Master_Log_Pos: 36238
               Relay_Log_File: localhost-relay-bin.000032
                Relay_Log_Pos: 36451
        Relay_Master_Log_File: mysql-bin.000014
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
```
### 测试回复

数据库被删除
```text
[root@localhost mysql]# mysql -uroot -p'_Jimb55!' -e 'drop database testb;show databases;'
mysql: [Warning] Using a password on the command line interface can be insecure.
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+

[root@localhost mysql]# mysql -uroot -p'_Jimb55!' -e 'show slave status\G;'
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.16.47.134
                  Master_User: salveruser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000014
          Read_Master_Log_Pos: 36614
               Relay_Log_File: localhost-relay-bin.000034
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000014
             Slave_IO_Running: Yes
            Slave_SQL_Running: No
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 1146
                   Last_Error: Error 'Table 'testb.gek_identifi_company' doesn't exist' on query. Default database: 'testb'. Query: 'insert into gek_identifi_company(name,sex,birthaddr) values('5b07cb8f2d39u','9',1527237520)'
```
发现从库 SQL 线程失败

```text
[root@localhost local]# mysql -uroot -p'_Jimb55!' -e 'stop slave;source /root/local/bf/all.dump;start slave;select sleep(1);show slave status\G;'     
mysql: [Warning] Using a password on the command line interface can be insecure.
+----------+
| sleep(1) |
+----------+
|        0 |
+----------+
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.16.47.134
                  Master_User: salveruser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000014
          Read_Master_Log_Pos: 36614
               Relay_Log_File: localhost-relay-bin.000040
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000014
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
```





## 复制整个data

### 操作

首先要关闭 mysql 服务，这个是因为这次拷贝的是整个data 文件夹，mysql 服务启动时会一直操作这个文件夹
```text
[root@localhost local]# service mysql stop
Redirecting to /bin/systemctl stop mysql.service
```

然后打包data 文件夹
```text
[root@localhost lib]# pwd
/var/lib
[root@localhost lib]# mkdir /var/lib/bf
[root@localhost lib]# tar -czvf mysql.tar mysql
```

最后便可以执行 `service mysql start` 来重启服务

### 测试
```text
[root@localhost lib]# rm -rf  mysql
[root@localhost lib]# service mysql start
Redirecting to /bin/systemctl start mysql.service
Job for mysqld.service failed because the control process exited with error code. See "systemctl status mysqld.service" and "journalctl -xe" for details.
```

操作简单直接，解压原来的压缩的就行了
```text
[root@localhost lib]# tar -zxvf mysql.tar
[root@localhost lib]# service mysql start
Redirecting to /bin/systemctl start mysql.service
[root@localhost lib]# mysql -uroot -p'_Jimb55' -e 'show slave status \G;'
mysql: [Warning] Using a password on the command line interface can be insecure.
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES)
[root@localhost lib]# mysql -uroot -p'_Jimb55!' -e 'show slave status \G;'
mysql: [Warning] Using a password on the command line interface can be insecure.
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.16.47.134
                  Master_User: salveruser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000014
          Read_Master_Log_Pos: 36614
               Relay_Log_File: localhost-relay-bin.000042
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000014
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
```

### 第三种是 Backing Up a Master or Slave by Making It Read Only
参考资料 [Backing Up a Master or Slave by Making It Read Only](https://dev.mysql.com/doc/refman/5.7/en/replication-solutions-backups-read-only.html)

