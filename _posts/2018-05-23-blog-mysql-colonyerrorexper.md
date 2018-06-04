---
title: 'mysql 集群错误实验'
date: 2018-05-23
tags:
  - linux
  - mysql
  - 架构
  - 实验
toc: true
toc_label: "目录"
header:
  teaser: /assets/images/teaser/mysql-colony.jpg
---

## 当主数据库挂掉后还原

### 问题的生产方法
环境
![Image text](/assets/images/blogs/mysql-colonyerrorexper/c1.jpg)

模拟脚本采用[模拟用户请求](/2018/05/25/blog-mysql-colonyerrorexper.html#模拟用户请求)
作用每访问一次，插入一条数据和查询一条数据
部署在 ubuntu 的机子，使用
```text
root@ubuntu:~# ab -n200 -c200 http://172.16.47.129/Exper/mysql/persion.php?rn=1 #进行并发访问
```

操作步骤
- 1 先执行上面语句
- 2 在上面语句还没执行完成时，关掉 数据库master 机子（直接关机）
- 3 重启 master 机子，查看插入数据库 和 查看ubuntu机子上的p.log(上面php脚本生产，主要查看最后录入数据的的总条目) 发现对上了，证明脚本录入数据库准确无误
- 4 再次执行第一步，ab
- 5 关掉master 机子 再次重启，发现 p.log 最后ID 和 master 机子表的ID 对上，数据无误

以上步骤模拟 master机子不堪并发挂掉，然而重启之后还是不堪并发而挂掉，只能关掉网站进行维护


### slave 问题
在 slave（172.16.47.132 server_id=2）查看slave状态
```text
 show slave status \G'
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.16.47.134
                  Master_User: salveruser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000011
          Read_Master_Log_Pos: 154
               Relay_Log_File: localhost-relay-bin.000015
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000008
             Slave_IO_Running: Yes
            Slave_SQL_Running: No
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 1062
                   Last_Error: Error 'Duplicate entry '743' for key 'PRIMARY'' on query. Default database: 'testb'. Query: 'INSERT INTO `gek_identifi_company` (`name`,`sex`,`birthaddr`) VALUES ('5b07e1103a0ae','9','1527243024')'
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 46714
              Relay_Log_Space: 826151
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 1062
               Last_SQL_Error: Error 'Duplicate entry '743' for key 'PRIMARY'' on query. Default database: 'testb'. Query: 'INSERT INTO `gek_identifi_company` (`name`,`sex`,`birthaddr`) VALUES ('5b07e1103a0ae','9','1527243024')'
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 1
                  Master_UUID: 95e423d0-5858-11e8-a7e9-000c29920633
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: 
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 180526 08:52:42
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
1 row in set (0.00 sec)
```

发现报了个错 Duplicate entry '743' for key 'PRIMARY'' 明显的，主键已经存在，查看了下数据库表 ，也真的的确存在
```text
# select * from gek_identifi_company;
| 743 | 5b07e1103a0ae | 9   | 1527243024 |
...
| 768 | 5b07e11c75e0f | 9   | 1527243036 |
| 769 | 5b07e11d7d718 | 9   | 1527243037 |
| 770 | 5b07e11d7defc | 9   | 1527243037 |
| 771 | 5b07e11e82c05 | 9   | 1527243038 |
| 772 | 5b07e11e82c0e | 9   | 1527243038 |
+-----+---------------+-----+------------+
772 rows in set (0.00 sec)
```

重启服务看看

```text
[root@localhost ~]# service  mysql restart
Redirecting to /bin/systemctl restart mysql.service
```

依然不行！

这时候查看 中继 localhost-relay-bin.000015 ，看看那个点为何错误

```
Slave_IO_State: Waiting for master to send event   当前状态
                  Master_Host: 172.16.47.134    master ip
                  Master_User: salveruser  
                  Master_Port: 3306  
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000011  当前master 使用 bin 文件
          Read_Master_Log_Pos: 154                对应的 post 点     
               Relay_Log_File: localhost-relay-bin.000015  当前relay-log
                Relay_Log_Pos: 320                         relay pos
        Relay_Master_Log_File: mysql-bin.000008          Relay 对应的 master binlog
             Slave_IO_Running: Yes                             
            Slave_SQL_Running: No
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 1062
                   Last_Error: Error 'Duplicate entry '743' for key 'PRIMARY'' on query. Default database: 'testb'. Query: 'INSERT INTO `gek_identifi_company` (`name`,`sex`,`birthaddr`) VALUES ('5b07e1103a0ae','9','1527243024')'
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 46714      失败的点（对应的 master binlog）
              Relay_Log_Space: 826151 
```

从上面信息得知，先查看 Slave_IO_Running 是 正常运行的
```text
[root@localhost ~]# ls -al /var/lib/mysql/
总用量 123856
drwxr-x---.  6 mysql mysql     4096 5月  26 17:19 .
drwxr-xr-x. 26 root  root      4096 5月  17 11:24 ..
-rw-r-----.  1 mysql mysql       56 5月  17 11:24 auto.cnf
-rw-r-----.  1 mysql mysql      443 5月  26 17:19 ib_buffer_pool
-rw-r-----.  1 mysql mysql 12582912 5月  26 17:19 ibdata1
-rw-r-----.  1 mysql mysql 50331648 5月  26 17:19 ib_logfile0
-rw-r-----.  1 mysql mysql 50331648 5月  17 11:24 ib_logfile1
-rw-r-----.  1 mysql mysql 12582912 5月  26 17:19 ibtmp1
-rw-r-----.  1 mysql mysql    68230 5月  26 17:19 localhost.localdomain.err
-rw-r-----.  1 mysql mysql        5 5月  26 17:19 localhost.localdomain.pid
-rw-r-----.  1 mysql mysql      211 5月  26 08:52 localhost-relay-bin.000014
-rw-r-----.  1 mysql mysql    12017 5月  26 08:52 localhost-relay-bin.000015
-rw-r-----.  1 mysql mysql   782244 5月  26 11:15 localhost-relay-bin.000016
-rw-r-----.  1 mysql mysql      377 5月  26 11:15 localhost-relay-bin.000017
-rw-r-----.  1 mysql mysql    30300 5月  26 11:55 localhost-relay-bin.000018
-rw-r-----.  1 mysql mysql      258 5月  26 11:55 localhost-relay-bin.000019
-rw-r-----.  1 mysql mysql      424 5月  26 17:14 localhost-relay-bin.000020
-rw-r-----.  1 mysql mysql      343 5月  26 17:16 localhost-relay-bin.000021
-rw-r-----.  1 mysql mysql      211 5月  26 17:16 localhost-relay-bin.000022
-rw-r-----.  1 mysql mysql      343 5月  26 17:19 localhost-relay-bin.000023
-rw-r-----.  1 mysql mysql      211 5月  26 17:19 localhost-relay-bin.000024
-rw-r-----.  1 mysql mysql      320 5月  26 17:19 localhost-relay-bin.000025
-rw-r-----.  1 mysql mysql      348 5月  26 17:19 localhost-relay-bin.index
-rw-r-----.  1 mysql mysql      136 5月  26 17:29 master.info
drwxr-x---.  2 mysql mysql     4096 5月  17 11:24 mysql
srwxrwxrwx.  1 mysql mysql        0 5月  26 17:19 mysql.sock
-rw-------.  1 mysql mysql        5 5月  26 17:19 mysql.sock.lock
drwxr-x---.  2 mysql mysql     8192 5月  17 11:24 performance_schema
-rw-r-----.  1 mysql mysql       69 5月  26 17:19 relay-log.info
drwxr-x---.  2 mysql mysql     8192 5月  17 11:24 sys
drwxr-x---.  2 mysql mysql      124 5月  25 16:30 testb
```

当前错误的是 发生在 localhost-relay-bin.000015 ，但 relay log 文件已经到000025 ，证明io线程还在不停的工作
所以问题锁定在 SQL 线程上，事实上上面status 已经清楚描述。

那么先到 localhost-relay-bin.000015 对应的 relay pos 点，查看
```text
[root@localhost ~]# /usr/local/es/mysql57/bin/mysqlbinlog  --base64-output=DECODE-ROWS   /var/lib/mysql/localhost-relay-bin.000015 | grep 320 -A50 
# at 320
#180526  2:10:17 server id 1  end_log_pos 46779 CRC32 0x8262455f        Anonymous_GTID  last_committed=120      sequence_number=121     rbr_only=no
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 385
#180526  2:10:17 server id 1  end_log_pos 46860 CRC32 0xb09998cd        Query   thread_id=125   exec_time=0     error_code=0
SET TIMESTAMP=1527271817/*!*/;
SET @@session.pseudo_thread_id=125/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1436549152/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C utf8 *//*!*/;
SET @@session.character_set_client=33,@@session.collation_connection=33,@@session.collation_server=33/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
BEGIN
/*!*/;
# at 466
# at 498
#180526  2:10:17 server id 1  end_log_pos 46892 CRC32 0x8ecaa7d1        Intvar
SET INSERT_ID=743/*!*/;   ### 这里面打印出了743
#180526  2:10:17 server id 1  end_log_pos 47071 CRC32 0x397fb622        Query   thread_id=125   exec_time=0     error_code=0
use `testb`/*!*/;
SET TIMESTAMP=1527271817/*!*/;
INSERT INTO `gek_identifi_company` (`name`,`sex`,`birthaddr`) VALUES ('5b07e1103a0ae','9','1527243024')
/*!*/;
# at 677
#180526  2:10:17 server id 1  end_log_pos 47102 CRC32 0x5f7082de        Xid = 386
COMMIT/*!*/;
# at 708
#180526  2:10:17 server id 1  end_log_pos 47167 CRC32 0xdc0b0c7a        Anonymous_GTID  last_committed=121      sequence_number=122     rbr_only=no
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 773
#180526  2:10:17 server id 1  end_log_pos 47248 CRC32 0xaa4f42c4        Query   thread_id=126   exec_time=0     error_code=0
SET TIMESTAMP=1527271817/*!*/;
BEGIN
/*!*/;
# at 854
# at 886
#180526  2:10:17 server id 1  end_log_pos 47280 CRC32 0x4e6ec79b        Intvar
SET INSERT_ID=744/*!*/;### 这里面打印出了 后面一直打印出ID
#180526  2:10:17 server id 1  end_log_pos 47459 CRC32 0x3854b82b        Query   thread_id=126   exec_time=0     error_code=0
SET TIMESTAMP=1527271817/*!*/;
```

查看文件localhost-relay-bin.000015 的最后一个 ID 录入 ,输出 /var/lib/mysql/localhost-relay-bin.000015 最后20行，发现 localhost-relay-bin.000015末尾最后录入的ID就是当前数据表的最大ID

```text
[root@localhost ~]# /usr/local/es/mysql57/bin/mysqlbinlog  --base64-output=DECODE-ROWS   /var/lib/mysql/localhost-relay-bin.000015 | tail -n 20
BEGIN
/*!*/;
# at 11718
# at 11750
#180526  2:10:31 server id 1  end_log_pos 58144 CRC32 0x4dfc54f3        Intvar
SET INSERT_ID=772/*!*/; #这是salve 数据库 的最后一个ID ，同时也是localhost-relay-bin.000015 文件的末尾
#180526  2:10:31 server id 1  end_log_pos 58323 CRC32 0xf9803bcb        Query   thread_id=154   exec_time=0     error_code=0
SET TIMESTAMP=1527271831/*!*/;
INSERT INTO `gek_identifi_company` (`name`,`sex`,`birthaddr`) VALUES ('5b07e11e82c0e','9','1527243038')
/*!*/;
# at 11929
#180526  2:10:31 server id 1  end_log_pos 58354 CRC32 0x81df93ae        Xid = 471
COMMIT/*!*/;
# at 11960 # 最后的 pos 点，pos点 下面将是文件末尾
#180526  8:52:42 server id 3  end_log_pos 12017 CRC32 0x0f6f63f4        Rotate to localhost-relay-bin.000016  pos: 4
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```

那么问题来了！既然localhost-relay-bin.000015 已经完整读完，那么，relay 的 pos 点为何是 320 而不是 11960

---
### 原因
???

### 解决办法
把 下面的两项矫正过来
```
Relay_Log_File: localhost-relay-bin.000015  当前relay-log
Relay_Log_Pos: 320                         relay pos
```
矫正到新的文件上，就是 localhost-relay-bin.000016

查看 localhost-relay-bin.000016 文件的开头前45行 ，可以很明显的看见  INSERT_ID=773 的ID，所以只要定位到这份文件对应的pos点便可
```text
[root@localhost ~]# /usr/local/es/mysql57/bin/mysqlbinlog  --base64-output=DECODE-ROWS   /var/lib/mysql/localhost-relay-bin.000016 | head -n 45
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 4   # localhost-relay-bin.000016  开头 pos 点
#180526  8:52:42 server id 3  end_log_pos 123 CRC32 0x98232281  Start: binlog v 4, server v 5.7.22 created 180526  8:52:42
# This Format_description_event appears in a relay log and was generated by the slave thread.
# at 123
#180526  8:52:42 server id 3  end_log_pos 154 CRC32 0x12b92834  Previous-GTIDs
# [empty]
# at 154
#700101  8:00:00 server id 1  end_log_pos 0 CRC32 0x6677862d    Rotate to mysql-bin.000009  pos: 4
# at 201
#180526 16:52:00 server id 1  end_log_pos 123 CRC32 0x30c49b38  Start: binlog v 4, server v 5.7.22-log created 180526 16:52:00 at startup
ROLLBACK/*!*/;
# at 320
#180526  8:52:42 server id 0  end_log_pos 367 CRC32 0x4cfe57cc  Rotate to mysql-bin.000009  pos: 154
# at 367
#180526 17:20:07 server id 1  end_log_pos 219 CRC32 0xec0490bc  Anonymous_GTID  last_committed=0        sequence_number=1       rbr_only=no
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 432
#180526 17:20:07 server id 1  end_log_pos 300 CRC32 0xfbdea217  Query   thread_id=5     exec_time=0     error_code=0
SET TIMESTAMP=1527326407/*!*/;
SET @@session.pseudo_thread_id=5/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1436549152/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C utf8 *//*!*/;
SET @@session.character_set_client=33,@@session.collation_connection=33,@@session.collation_server=33/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
BEGIN
/*!*/;
# at 513
# at 545
#180526 17:20:07 server id 1  end_log_pos 332 CRC32 0x6b8b3c67  Intvar
SET INSERT_ID=773/*!*/;  # iD 773，正确的ID ，数据表当前最大ID 是772
#180526 17:20:07 server id 1  end_log_pos 511 CRC32 0x032f3a07  Query   thread_id=5     exec_time=0     error_code=0
use `testb`/*!*/;
SET TIMESTAMP=1527326407/*!*/;
INSERT INTO `gek_identifi_company` (`name`,`sex`,`birthaddr`) VALUES ('5b08b6502153b','9','1527297616')
/*!*/;
# at 724
#180526 17:20:07 server id 1  end_log_pos 542 CRC32 0xa6b5329c  Xid = 34
COMMIT/*!*/;
# at 755
```


先 stop slave;服务

再打开 relay-log.info ,跟master.info 原理一样,参考资料[slave-logs-status](https://dev.mysql.com/doc/refman/5.7/en/slave-logs-status.html)
```text
# vi /var/lib/mysql/relay-log.info 
7
./localhost-relay-bin.000015
320
mysql-bin.000008
46714
0
0
1

### 修改为 
7
./localhost-relay-bin.000016
4
#其他的呢？鬼知什么东西，全删了

```

重启服务
```text
[root@localhost ~]# service mysql restart
Redirecting to /bin/systemctl restart mysql.service
[root@localhost ~]# mysql
mysql> start slave;
Query OK, 0 rows affected, 1 warning (0.00 sec)
mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.16.47.134
                  Master_User: salveruser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000011
          Read_Master_Log_Pos: 154
               Relay_Log_File: localhost-relay-bin.000027
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000011
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
 ...
mysql> select * from gek_identifi_company;
| 2864 | 5b08d180a104d | 9   | 1527304576 |
+------+---------------+-----+------------+
2864 rows in set (0.01 sec) 
```
成功还原对的点,数据自动同步!
再次测试master端插入数据，slave端同步没问题
```text
| 2865 | 5b08d180a1042 | 9   | 1527304577 |
+------+---------------+-----+------------+
```
> PS: The next time the slave starts up, it reads the two logs to determine how far it has proceeded in reading binary logs from the master and in processing its own relay logs.
> 官方原文  [slave-logs-status](https://dev.mysql.com/doc/refman/5.7/en/slave-logs-status.html)

### 便捷操作
```text
mysql>stop slave;
mysql>exit;
# vi /var/lib/mysql/relay-log.info 
7
./localhost-relay-bin.000015
320
mysql-bin.000008
46714
0
0
1
#改为
7
./localhost-relay-bin.000016
4

#service mysql restart
#mysql -uroot -p'_Jimb55!'
mysql> start slave;
```



## data文件夹新添加slave
当使用拷贝整个旧slave data文件夹 来作为新添加slave的库时
材料 master slave2（旧）  slave3（新）

首先，停掉服务（必须的，保证同步过程不出现数据更改）
slave2（旧） 操作
```text
[root@localhost lib]# service mysql stop
[root@localhost lib]# pwd
/var/lib
[root@localhost lib]# tar -zcvf mysql.tar mysql
[root@localhost lib]# service mysql start
```
slave3（新） 操作
```text
[root@localhost lib]# pwd
/var/lib
[root@localhost lib]# scp  root@172.16.47.140:/var/lib/mysql.tar  mysql.tar
[root@localhost lib]# tar -zxvf mysql.tar 
[root@localhost lib]# vi /etc/my.cnf
# 修改 my.cnf 
servier_id = 3;

[root@localhost lib]# service mysql start
[root@localhost lib]# mysql -uroot -p'_Jimb55!'
mysql> show slave status;
```
发现一切正常

但当打开 slave2 时发现
报了如下的错
```text
  Last_IO_Errno: 1236
  Last_IO_Error: Got fatal error 1236 from master when reading data from binary log: 'A slave with the same server_uuid/server_id as this slave has connected to the master; the first event 'mysql-bin.000016' at 154, the last event read from './mysql-bin.000016' at 123, the last byte read from './mysql-bin.000016' at 154.'
```
重启一下 `stop slave` ， `start slave `发现又正常了，结果看了下slave3（新）的
```text
  Last_IO_Errno: 1236
  Last_IO_Error: Got fatal error 1236 from master when reading data from binary log: 'A slave with the same server_uuid/server_id as this slave has connected to the master; the first event 'mysql-bin.000016' at 154, the last event read from './mysql-bin.000016' at 123, the last byte read from './mysql-bin.000016' at 154.'
```
报的错跟上面的一模一样

到master 上看了下错误日志
```text
2018-06-01T14:11:07.161773Z 9 [Note] Start binlog_dump to master_thread_id(9) slave_server(3), pos(mysql-bin.000015, 930)
2018-06-01T14:13:30.001582Z 7 [Note] Slave for channel '': received end packet from server due to dump thread being killed on master. Dump threads are killed for example during master shutdown, explicitly by a user, or when the master receives a binlog send request from a duplicate server UUID <95e423d0-5858-11e8-a7e9-000c29920633> : Error 
2018-06-01T14:13:30.001714Z 7 [Note] Slave I/O thread: Failed reading log event, reconnecting to retry, log 'mysql-bin.000001' at position 2393 for channel ''
2018-06-01T14:13:30.001841Z 7 [Warning] Storing MySQL user name or password information in the master info repository is not secure and is therefore not recommended. Please consider using the USER and PASSWORD connection options for START SLAVE; see the 'START SLAVE Syntax' in the MySQL Manual for more information.
2018-06-01T14:13:30.018831Z 7 [ERROR] Slave I/O for channel '': error reconnecting to master 'salveruser@172.16.47.140:3306' - retry-time: 60  retries: 1, Error_code: 2003
2018-06-01T14:13:38.490274Z 11 [Note] While initializing dump thread for slave with UUID <c2e1598d-5981-11e8-a10c-000c2986eda0>, found a zombie dump thread with the same UUID. Master is killing the zombie dump thread(9).
2018-06-01T14:13:38.490421Z 11 [Note] Start binlog_dump to master_thread_id(11) slave_server(2), pos(mysql-bin.000015, 2482)
2018-06-01T14:14:30.075264Z 7 [Note] Slave for channel '': connected to master 'salveruser@172.16.47.140:3306',replication resumed in log 'mysql-bin.000001' at position 2393
2018-06-01T14:54:34.659545Z 12 [Note] While initializing dump thread for slave with UUID <c2e1598d-5981-11e8-a10c-000c2986eda0>, found a zombie dump thread with the same UUID. Master is killing the zombie dump thread(11).
2018-06-01T14:54:34.660290Z 12 [Note] Start binlog_dump to master_thread_id(12) slave_server(3), pos(mysql-bin.000016, 154)
2018-06-01T14:56:41.879315Z 14 [Note] While initializing dump thread for slave with UUID <c2e1598d-5981-11e8-a10c-000c2986eda0>, found a zombie dump thread with the same UUID. Master is killing the zombie dump thread(12).
2018-06-01T14:56:41.879408Z 14 [Note] Start binlog_dump to master_thread_id(14) slave_server(2), pos(mysql-bin.000016, 154)
2018-06-01T15:01:28.370304Z 0 [Note] InnoDB: page_cleaner: 1000ms intended loop took 5383ms. The settings might not be optimal. (flushed=0 and evicted=0, during the time.)


mysql> SHOW SLAVE HOSTS;
+-----------+---------------+------+-----------+--------------------------------------+
| Server_id | Host          | Port | Master_id | Slave_UUID                           |
+-----------+---------------+------+-----------+--------------------------------------+
|         2 | 172.16.47.136 | 3306 |         1 | c2e1598d-5981-11e8-a10c-000c2986eda0 |
+-----------+---------------+------+-----------+--------------------------------------+
1 row in set (0.00 sec)
```
发现只有一台机子有效


```text
slave 3
mysql> show variables like '%server%id%';
+----------------+--------------------------------------+
| Variable_name  | Value                                |
+----------------+--------------------------------------+
| server_id      | 3                                    |
| server_id_bits | 32                                   |
| server_uuid    | c2e1598d-5981-11e8-a10c-000c2986eda0 |
+----------------+--------------------------------------+

slave 2
mysql> show variables like '%server%id%';
+----------------+--------------------------------------+
| Variable_name  | Value                                |
+----------------+--------------------------------------+
| server_id      | 2                                    |
| server_id_bits | 32                                   |
| server_uuid    | c2e1598d-5981-11e8-a10c-000c2986eda0 |
+----------------+--------------------------------------+
```
发现 其 uuid 居然相同，估计是 data 某份文件记录相同，导致拷贝过来发生冲突了，到网上一查，果真如此
[slave have equal MySQL server UUIDs](https://yq.aliyun.com/ziliao/22664)
[双slave的server_uuid相同问题](https://blog.csdn.net/dba_waterbin/article/details/27533869)

slave3（新） 操作
```text
[root@localhost mysql]# mv auto.cnf auto.cnf.bf
[root@localhost mysql]# service mysql restart   
Redirecting to /bin/systemctl restart mysql.service
```

> auto.cnf 就是用来记录 

master 操作 server UUID 的
```text
mysql> SHOW SLAVE HOSTS;
+-----------+---------------+------+-----------+--------------------------------------+
| Server_id | Host          | Port | Master_id | Slave_UUID                           |
+-----------+---------------+------+-----------+--------------------------------------+
|         3 | 172.16.47.132 | 3306 |         1 | 2aa3ee59-656f-11e8-a977-000c2991d93f |
|         2 | 172.16.47.136 | 3306 |         1 | c2e1598d-5981-11e8-a10c-000c2986eda0 |
+-----------+---------------+------+-----------+--------------------------------------+
2 rows in set (0.00 sec)
```

### 便捷操作
```text
# slave2
[root@localhost lib]# service mysql stop
[root@localhost lib]# pwd
/var/lib
[root@localhost lib]# tar -zcvf mysql.tar mysql
[root@localhost lib]# service mysql start


# slave3
[root@localhost lib]# pwd
/var/lib
[root@localhost lib]# scp  root@172.16.47.140:/var/lib/mysql.tar  mysql.tar
[root@localhost lib]# tar -zxvf mysql.tar 
[root@localhost lib]# vi /etc/my.cnf
# 修改 my.cnf 
servier_id = 3;
[root@localhost lib]# cd mysql
[root@localhost mysql]# mv auto.cnf auto.cnf.bf
[root@localhost lib]# service mysql start
[root@localhost lib]# mysql -uroot -p'_Jimb55!'
mysql> start slave;

```