---
title: 'zabbix 知识或问题汇总'
date: 2018-06-09
tags:
  - linux
  - zabbix
toc: true
toc_label: "目录"
header:
  teaser: /assets/images/teaser/zabbix.png
---

## 更换中文 

![Image text](/assets/images/blogs/zabbix-other/z0.png)

但这时的图标下文字会显示问题，需要上传字体
```text
[root@localhost fonts]# pwd
/var/www/html/fonts
[root@localhost fonts]# ls -al
总用量 16128
drwxr-xr-x.  2 root root       66 6月  13 15:01 .
drwxr-xr-x. 14 root root     4096 6月  12 17:47 ..
-rw-r--r--.  1 root root   756072 6月   7 09:02 DejaVuSans.ttf
-rw-r--r--.  1 root root  4051204 5月  14 09:29 STXINWEI.ttf
```

![Image text](/assets/images/blogs/zabbix-other/z0.png)

再修改 文件 

```text
[root@localhost fonts]# vi /var/www/html/include/defines.inc.php 
...
define('ZBX_FONT_NAME', 'STXINWEI');
define('ZBX_GRAPH_FONT_NAME','STXINWEI');
...
```

参考资料: [zabbix 3.2.2 server web展示如何显示中文](https://www.cnblogs.com/miclesvic/p/6145171.html)


## agent 启动失败

[root@localhost data]# /etc/init.d/zabbix_agentd start
zabbix_agentd [1838]: user zabbix does not exist
zabbix_agentd [1838]: cannot run as root!
Zabbix agent started.

没有发现用户zabbix 问题
添加用户,再次重启即可解决问题

```text
[root@localhost local]# groupadd  zabbix
[root@localhost local]# useradd  -g  zabbix  -s  /sbin/nologin  zabbix
```

## 链接 agent 失败

Get value from agent failed: cannot connect to [[172.16.47.138]:10050]: [113] No route to host

检查 agent 是否开启成功，检查10050 10051 端口是否开齐

```text
[root@localhost data]# ps -ef | grep zabbix           
zabbix     1953      1  0 16:35 ?        00:00:00 /usr/local/sbin/zabbix_agentd
zabbix     1954   1953  0 16:35 ?        00:00:00 /usr/local/sbin/zabbix_agentd: collector [idle 1 sec]
zabbix     1955   1953  0 16:35 ?        00:00:00 /usr/local/sbin/zabbix_agentd: listener #1 [waiting for connection]
zabbix     1956   1953  0 16:35 ?        00:00:00 /usr/local/sbin/zabbix_agentd: listener #2 [waiting for connection]
zabbix     1957   1953  0 16:35 ?        00:00:00 /usr/local/sbin/zabbix_agentd: listener #3 [waiting for connection]
zabbix     1958   1953  0 16:35 ?        00:00:00 /usr/local/sbin/zabbix_agentd: active checks #1 [idle 1 sec]
root       1960   1436  0 16:35 pts/0    00:00:00 grep --color=auto zabbix

[root@localhost data]# firewall-cmd --zone=public --add-port=10050/tcp --permanent  
success
[root@localhost data]# firewall-cmd --zone=public --add-port=10051/tcp --permanent 
success
[root@localhost data]# firewall-cmd --reload
success
[root@localhost data]# firewall-cmd --list-ports
80/tcp 3306/tcp 10050/tcp 10051/tcp
```

## 关于zabbix key 脚本找不到 mysql.sock 问题

[root@localhost local]# /usr/local/zabbix/bin/zabbix_get -s 172.16.47.138 -k mysql.replication
ERROR 2002 (HY000): Can't connect to local MySQL server through socket '/var/lib/mysql/mysql.sock' (13)
0

问题本质抽成一下，第一是目录权限的问题,**zabbix_agent**是使用**zabbix**用户执行操作的，没有读取 `/var/lib/mysql/mysql.sock`的权限

![Image text](/assets/images/blogs/zabbix-other/sock1.png)

![Image text](/assets/images/blogs/zabbix-other/sock2.png)

[mysql终端链接问题](/2018/06/08/blog-mysql-other.html#关于mysql终端链接mysql.server问题)
[参考资料](https://www.cnblogs.com/mrwang1101/p/4887842.html)