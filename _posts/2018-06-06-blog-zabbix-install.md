---
title: 'zabbix 安装'
date: 2018-06-06
tags:
  - linux
  - mysql
toc: true
toc_label: "目录"
header:
  teaser: /assets/images/blogs/mysql-install/mysql-install.png
---

## zabbix 3.2 server安装

### 安装mysql

[ mysql5.5 安装 ](http://jimb.net/2018/05/14/blog-mysql-install.html#mysql55-%E7%BC%96%E8%AF%91%E5%AE%89%E8%A3%85)



### 安装zabbix
```text
[root@localhost local]# wget  http://sourceforge.net/projects/zabbix/files/ZABBIX%20Latest%20Stable/3.2.6/zabbix-3.2.6.tar.gz
[root@localhost local]# yum -y  install  curl  curl-devel net-snmp net-snmp-devel perl-DBI net-snmp-devel
[root@localhost local]# groupadd  zabbix
[root@localhost local]# useradd  -g  zabbix  -s  /sbin/nologin  zabbix
[root@localhost local]# mysql -uroot -p'_Jimb55!'
 mysql> create  database  zabbix  charset=utf8;
 Query OK, 0 rows affected (0.03 sec)
 
 mysql> grant all on zabbix.* to zabbix@'localhost' identified by '123456';
 Query OK, 0 rows affected (0.03 sec)
 
 mysql> flush privileges;
 Query OK, 0 rows affected (0.00 sec)
 
 mysql> exit;
 
[root@localhost local]# tar -zxvf zabbix-3.2.6.tar.gz 
[root@localhost local]# cd zabbix-3.2.6
[root@localhost zabbix-3.2.6]# mysql -uzabbix -p123456 zabbix <database/mysql/schema.sql
[root@localhost zabbix-3.2.6]# mysql -uzabbix -p123456 zabbix <database/mysql/images.sql
[root@localhost zabbix-3.2.6]# mysql -uzabbix -p123456 zabbix < database/mysql/data.sql

[root@localhost zabbix-3.2.6]# ./configure --prefix=/usr/local/zabbix/ --enable-server --enable-agent --with-mysql=/usr/local/mysql55/bin/mysql_config --enable-ipv6 --with-net-snmp  --with-libcurl
[root@localhost zabbix-3.2.6]# make && make install

[root@localhost zabbix-3.2.6]# ln -s /usr/local/zabbix/sbin/zabbix_*  /usr/local/sbin/

[root@localhost zabbix-3.2.6]# cd /usr/local/zabbix/etc/
[root@localhost etc]# cp  zabbix_server.conf  zabbix_server.conf.bak
[root@localhost etc]# vi  zabbix_server.conf
LogFile=/tmp/zabbix_server.log
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=123456

[root@localhost etc]# cd ~/local/zabbix-3.2.6
[root@localhost zabbix-3.2.6]# cp  misc/init.d/tru64/zabbix_server  /etc/init.d/zabbix_server
[root@localhost zabbix-3.2.6]# chmod  o+x  /etc/init.d/zabbix_server
[root@localhost zabbix-3.2.6]# /etc/init.d/zabbix_server start
Zabbix server started.
```
> ps：注意 mysql_config 的地址指向 安装目录下的bin/mysql_config

以上，zabbix 安装完毕 ，还需要个服务器打开 zabbix web 服务

### 安装apache和php 环境
```text
[root@localhost zabbix-3.2.6]# yum install  php.x86_64  mysql-devel php-mysql.x86_64 -y  --skip-broken
[root@localhost zabbix-3.2.6]# yum install curl curl-devel net-snmp net-snmp-devel perl-DBI php-gd php-xml php-bcmath php-mbstring  gd  gd-devel -y
[root@localhost zabbix-3.2.6]# pwd
/root/local/zabbix-3.2.6
[root@localhost zabbix-3.2.6]# cp -rf frontends/php/*    /var/www/html/  

[root@localhost zabbix-3.2.6]# mkdir -p /var/lib/mysql/
[root@localhost zabbix-3.2.6]# ln -s /tmp/mysql.sock  /var/lib/mysql/
[root@localhost zabbix-3.2.6]# sed   -i '/post_max_size/s/8/16/g;/max_execution_time/s/30/300/g;/max_input_time/s/60/300/g;s/\;date.timezone.*/date.timezone \= PRC/g;s/\;always_populate_raw_post_data/always_populate_raw_post_data/g'  /etc/php.ini
```

创建zabbix 的 mysql 账户
```text
[root@localhost zabbix-3.2.6]# mysql -uroot -p'_Jimb55!'    
mysql> grant all on zabbix.* to zabbix@'localhost' identified by '123456';  
Query OK, 0 rows affected (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)
```

启动环境
```text
[root@localhost zabbix-3.2.6]# apachectl start
[root@localhost zabbix-3.2.6]# /etc/init.d/zabbix_server  restart
Zabbix server started.
[root@localhost zabbix-3.2.6]# service mysql restart
Redirecting to /bin/systemctl restart mysql.service
[root@localhost zabbix-3.2.6]# 
```

开启防火墙 80 端口
```text
[root@localhost zabbix-3.2.6]# firewall-cmd --zone=public --add-port=80/tcp --permanent
success
[root@localhost zabbix-3.2.6]# firewall-cmd --reload
success
[root@localhost zabbix-3.2.6]# firewall-cmd --list-ports
80/tcp
```

然后一直填写资料

![Image text](/assets/images/blogs/zabbix_install/zb-c.png)

这里需要添加配置文件，可以直接下载
下载文件上传到服务器指定位置，当然拷贝内容更容易，完成后显示

```text
Unable to overwrite the existing configuration file
```

看似是报错，但不用理会，直接打开 http://zabbix.jimbtest.com/index.php

当然，以上做法过于麻烦，可以直接给 `conf` 目录权限
```text
[root@localhost html]# chown -R apache conf
```
刷新页面，显示绿色的字体
```text
Congratulations! You have successfully installed Zabbix frontend.
```
按下finish!

登录账号密码初始为 **admin** : **zabbix**


## zabbix 3.2 agent安装
```text
[root@localhost local]# wget  http://sourceforge.net/projects/zabbix/files/ZABBIX%20Latest%20Stable/3.2.6/zabbix-3.2.6.tar.gz
[root@localhost local]# yum install -y gcc c ncurses-devel cmake libaio bison gcc-c++  git   ncurses curl  curl-devel net-snmp net-snmp-devel perl-DBI net-snmp-devel
[root@localhost local]# tar -zxvf zabbix-3.2.6.tar.gz
[root@localhost local]# cd zabbix-3.2.6
[root@localhost zabbix-3.2.6]# ./configure  --prefix=/usr/local/zabbix  --enable-agent

[root@localhost zabbix-3.2.6]# make && make install
[root@localhost zabbix-3.2.6]# ln  -s  /usr/local/zabbix/sbin/zabbix_*  /usr/local/sbin/

[root@localhost zabbix-3.2.6]# cd /usr/local/zabbix/etc/
[root@localhost etc]# ls -al
总用量 12
drwxr-xr-x. 3 root root    60 6月   7 10:55 .
drwxr-xr-x. 7 root root    64 6月   7 10:55 ..
-rw-r--r--. 1 root root 10234 6月   7 10:55 zabbix_agentd.conf
drwxr-xr-x. 2 root root     6 6月   7 10:55 zabbix_agentd.conf.d
[root@localhost etc]# vi zabbix_agentd.conf
...
Server=172.16.47.142
ServerActive=172.16.47.142
Hostname = 172.16.47.143
...

[root@localhost etc]# cd ~/local/zabbix-3.2.6
[root@localhost zabbix-3.2.6]# cp misc/init.d/tru64/zabbix_agentd /etc/init.d/zabbix_agentd
[root@localhost zabbix-3.2.6]# chmod o+x /etc/init.d/zabbix_agentd
[root@localhost zabbix-3.2.6]# groupadd  zabbix
[root@localhost zabbix-3.2.6]# useradd -g zabbix -s   /sbin/nologin  zabbix

[root@localhost zabbix-3.2.6]# /etc/init.d/zabbix_agentd  start
```

接着就是添加配置文件，让zabbix server 找到 agent 端

![Image text](/assets/images/blogs/zabbix_install/zs_c1.png)

![Image text](/assets/images/blogs/zabbix_install/zs_c2.png)

![Image text](/assets/images/blogs/zabbix_install/zs_c3.png)

以上没有问题，但下面正确来说，是应该呈现绿色的，表示正常的链接状态，好明显下面链接不到
过段时间甚至变成了红色，打印出报错信息

![Image text](/assets/images/blogs/zabbix_install/zs_c4.png)


使用 **zabbix_get** 查看是否能取得 agent 端的机子内容
```text
[root@localhost html]# /usr/local/zabbix/bin/zabbix_get  -s  172.16.47.143   -k  system.uname 
zabbix_get [57934]: Get value error: cannot connect to [[172.16.47.143]:10050]: [113] No route to host
或
一直等待 ，这都是链接不成功引起的问题
```
那么就应该考虑是否是防火墙问题，或配置文件（`/usr/local/zabbix/etc/zabbix_agentd.conf`）没写对

开启防火墙(**agent** 端)
```text
[root@localhost zabbix-3.2.6]# firewall-cmd --zone=public --add-port=10050/tcp --permanent 
success
[root@localhost zabbix-3.2.6]# firewall-cmd --reload
success
[root@localhost zabbix-3.2.6]# firewall-cmd --list-ports
10050/tcp
```
为啥是 10050 ？ 是上面的配置文件配置的 Agent interfaces
server 端 再次测试一下 OK~!
```text
[root@localhost html]# /usr/local/zabbix/bin/zabbix_get  -s  172.16.47.143   -k  system.uname
Linux localhost.localdomain 3.10.0-693.el7.x86_64 #1 SMP Tue Aug 22 21:09:27 UTC 2017 x86_64
```

回到zabbix web 查看 ,  Reset 一下! OK~!

![Image text](/assets/images/blogs/zabbix_install/zs_c5.png)

![Image text](/assets/images/blogs/zabbix_install/zs_c6.png)
