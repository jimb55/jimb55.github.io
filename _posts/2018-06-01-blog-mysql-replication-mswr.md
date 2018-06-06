---
title: 'mysql Replication 主从同步'
date: 2018-06-01
tags:
  - linux
  - mysql
toc: true
toc_label: "目录"
header:
  teaser: /assets/images/blogs/mysql-install/mysql-install.png
---

## 数据库读写分离

读写分离 实现有两种方法
 - 一是使用第三方插件（mysql-proxy，mycat等）
 - 二是使用代码实现（逻辑实现）
 
### 使用mysql-proxy

mysql 的 读写分离中间件 原理
![Image text](/assets/images/blogs/mysql-mswr/mswr1.jpg)

#### 安装
```text
[root@localhost local]# wget http://ftp.ntu.edu.tw/pub/MySQL/Downloads/MySQL-Proxy/mysql-proxy-0.8.4-linux-el6-x86-64bit.tar.gz  
[root@localhost local]# useradd  -r  mysql-proxy
[root@localhost local]# tar  zxvf  mysql-proxy-0.8.4-linux-el6-x86-64bit.tar.gz  -C  /usr/local
[root@localhost local]# mv  /usr/local/mysql-proxy-0.8.4-linux-el6-x86-64bit  /usr/local/mysql-proxy
[root@localhost local]# ln -s /usr/local/mysql-proxy/bin/mysql-proxy /usr/bin/
```


```text
mysql-proxy --daemon --log-level=debug --user=mysql-proxy --keepalive \
            --proxy-address=172.16.47.137:3306 \
            --log-file=/var/log/mysql-proxy.log \
            --plugins="proxy" \
            --proxy-backend-addresses="172.16.47.134:3306" \
            --proxy-read-only-backend-addresses="172.16.47.140:3306" \
            --proxy-read-only-backend-addresses="172.16.47.141:3306" \
            --proxy-lua-script="/usr/local/mysql-proxy/share/doc/mysql-proxy/rw-splitting.lua" \
            --plugins=admin \
            --admin-username="admin" \
            --admin-password="admin" \
            --admin-lua-script="/usr/local/mysql-proxy/lib/mysql-proxy/lua/admin.lua"
```

- help-all ：获取全部帮助信息；
- proxy-address=host:port ：代理服务监听的地址和端口，默认为4040；
- admin-address=host:port ：管理模块监听的地址和端口，默认为4041；
- proxy-backend-addresses=host:port ：后端mysql服务器的地址和端口；
- proxy-read-only-backend-addresses=host:port ：后端只读mysql服务器的地址和端口；
- proxy-lua-script=file_name ：完成mysql代理功能的Lua脚本；
- daemon ：以守护进程模式启动mysql-proxy；
- keepalive ：在mysql-proxy崩溃时尝试重启之；
- log-file=/path/to/log_file_name ：日志文件名称；
- log-level=level ：日志级别；
- log-use-syslog ：基于syslog记录日志；
- plugins=plugin ：在mysql-proxy启动时加载的插件；
- user=user_name ：运行mysql-proxy进程的用户；
- defaults-file=/path/to/conf_file_name ：默认使用的配置文件路径，其配置段使用[mysql-proxy]标识；
- proxy-skip-profiling ：禁用profile；
- pid-file=/path/to/pid_file_name ：进程文件名；

查看开启状态
```text
[root@localhost bin]# netstat -tnpl | grep mysql 
tcp        0      0 0.0.0.0:4040            0.0.0.0:*               LISTEN      4995/mysql-proxy    
tcp        0      0 0.0.0.0:4041            0.0.0.0:*               LISTEN      4995/mysql-proxy   
```

登录随便一台拥有mysql 终端的机子,使用 mysql终端登录
```text
mysql  -h172.16.47.137  -uadmin  -padmin  -P 4041
```

若是报错
```text
[root@localhost mysql]# mysql  -h172.16.47.137  -uadmin  -padmin  -P 4041
mysql: [Warning] Using a password on the command line interface can be insecure.
ERROR 2003 (HY000): Can't connect to MySQL server on '172.16.47.137' (113)
```
[服务器防火墙开启](/2018/05/16/blog-mysql-replication-mse.html#2-%E9%93%BE%E6%8E%A5%E4%B8%BB%E5%BA%93%E5%A4%B1%E8%B4%A5)

---
同时确保 master slave2 slave3 防火墙开启
```text
[root@localhost ~]# firewall-cmd --list-ports
80/tcp 3306/tcp
```
拥有能访问的 账户 与权限
```text
mysql> use mysql;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> select Host,User from user; 
+-----------+---------------+
| Host      | User          |
+-----------+---------------+
| %         | allperson     |
| %         | salveruser    |
| %         | tongbu        |
| localhost | mysql.session |
| localhost | mysql.sys     |
| localhost | root          |
+-----------+---------------+
6 rows in set (0.00 sec)
```
---

```text
mysql> SELECT * FROM backends;
+-------------+--------------------+---------+------+------+-------------------+
| backend_ndx | address            | state   | type | uuid | connected_clients |
+-------------+--------------------+---------+------+------+-------------------+
|           1 | 172.16.47.134:3306 | unknown | rw   | NULL |                 0 |
|           2 | 172.16.47.140:3306 | unknown | ro   | NULL |                 0 |
|           3 | 172.16.47.141:3306 | unknown | ro   | NULL |                 0 |
+-------------+--------------------+---------+------+------+-------------------+
3 rows in set (0.00 sec)
```

发现全部状态都是 `unknown`,这是 它的机制，当达到某次数访问机制才会启动，查询多此时就能解决这个问题，当然也可以修改配置文件

```text
vi /usr/local/mysql-proxy/share/doc/mysql-proxy/rw-splitting.lua

# 修改
if not proxy.global.config.rwsplit then
        proxy.global.config.rwsplit = {
                min_idle_connections = 1,
                max_idle_connections = 1,

                is_debug = false
        }
```

尝试运行**show databases**
```text
[root@localhost bin]# mysql  -h172.16.47.137  -uallperson  -p123456  -P 3306 -e ' show databases;'

[root@localhost bin]# mysql  -h172.16.47.137  -uadmin  -padmin  -P 4041  
MySQL [(none)]> SELECT * FROM backends
    -> ;
+-------------+--------------------+-------+------+------+-------------------+
| backend_ndx | address            | state | type | uuid | connected_clients |
+-------------+--------------------+-------+------+------+-------------------+
|           1 | 172.16.47.134:3306 | up    | rw   | NULL |                 0 |
|           2 | 172.16.47.138:3306 | up    | ro   | NULL |                 0 |
|           3 | 172.16.47.141:3306 | up    | ro   | NULL |                 0 |
+-------------+--------------------+-------+------+------+-------------------+
3 rows in set (0.00 sec)
```


### 代码实现