---
title: 'mysql Replication 双主'
date: 2018-05-30
tags:
  - linux
  - mysql
toc: true
toc_label: "目录"
header:
  teaser: /assets/images/blogs/mysql-install/mysql-install.png
---

双主的作用除可能减轻 写操作，还能使系统高可用，一台master 崩了，切换到另一台


## 多主多从
![Image text](/assets/images/blogs/mysql-twomaster/twomaster1.jpg)

先准备一个主从系统，双主的原理就是某两台机互为主从

首先在master机子的 `/etc/my.cnf`上添加如下代码
```text
[mysqld]
auto_increment_offset = 1
auto_increment_increment = 2
```

然后在 其中选定一台salve机作为master2
配置文件`/etc/my.cnf`添加代码
```text
server-id =2  

#开启 bin log ，作为master
log-bin=mysql-bin  
binlog_format=mixed

# id 自增
auto_increment_offset = 2
auto_increment_increment = 2
```
`auto_increment_increment`，`auto_increment_offset` 这两个主要是处理表的id自增问题，使其不冲突

重启mysql 服务便可! 配置就完成了

[服务器防火墙开启](/2018/05/16/blog-mysql-replication-mse.html#2-%E9%93%BE%E6%8E%A5%E4%B8%BB%E5%BA%93%E5%A4%B1%E8%B4%A5)

剩下的就是在 master2(原来的slave1) 上创建 角色
```text
mysql> grant  replication  slave  on *.* to  'salveruser'@'%'  identified by  '123456';
mysql> show master status;
       +------------------+----------+--------------+------------------+-------------------+
       | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
       +------------------+----------+--------------+------------------+-------------------+
       | mysql-bin.000001 |     453  |              |                  |                   |
       +------------------+----------+--------------+------------------+-------------------+
       1 row in set (0.00 sec)
```

在master1（原来的master） 上执行认主操作

```text
# change master to master_host='172.16.47.140', master_user='salveruser',master_password='123456',master_log_file='mysql-bin.000001',master_log_pos=453;
# start slave;
```


## 多源 多从
![Image text](/assets/images/blogs/mysql-twomaster/twomaster2.jpg)


配置请参考:[MySQL 5.7 的多源复制](https://www.oschina.net/translate/mysql-5-7-multi-source-replication)

