---
title: 'zabbix 触发器监控'
date: 2018-06-13
tags:
  - linux
  - zabbix
toc: true
toc_label: "目录"
header:
  teaser: /assets/images/teaser/zabbix.png
---

## 自定义监控项目

使用触发器监控除 cpu,内存,swap以外的服务。例如 mysql，nginx,redis 等服务

### 开启 nginx stub_status 模块

需要**nginx**安装 **--with-http_stub_status_module** 模块,用于取得当前nginx的状态
```text
./configure --user=www --group=www --prefix=/usr/local/nginx --with-http_stub_status_module
```

配置
```text
location = /nginx-status {
      stub_status on;
      access_log  off;
      allow 127.0.0.1;
      allow 172.16.47.142; 
      #### zabbix服务器端的IP地址一般为内网IP
}
```
访问
```text
[root@localhost nginx]# curl 172.16.47.138/nginx-status
Active connections: 1 
server accepts handled requests
 2 2 2 
Reading: 0 Writing: 1 Waiting: 0 
```

### 监控 nginx

首先需要添加 **zabbix_agent** ，部署好后，添加监控脚本和写入监控项

shell 脚本来自[https://github.com/thecamels/zabbix/blob/master/bin/nginx-check.sh](https://github.com/thecamels/zabbix/blob/master/bin/nginx-check.sh)
```
#!/bin/bash
##################################
# Zabbix monitoring script
#
# nginx:
#  - anything available via nginx stub-status module
#
##################################
# Contact:
#  vincent.viallet@gmail.com
##################################
# ChangeLog:
#  20100922	VV	initial creation
##################################

# Zabbix requested parameter
ZBX_REQ_DATA="$1"
ZBX_REQ_DATA_URL="$2"

# Nginx defaults
NGINX_STATUS_DEFAULT_URL="http://172.16.47.138/nginx-status"
WGET_BIN="/usr/bin/wget"

#
# Error handling:
#  - need to be displayable in Zabbix (avoid NOT_SUPPORTED)
#  - items need to be of type "float" (allow negative + float)
#
ERROR_NO_ACCESS_FILE="-0.9900"
ERROR_NO_ACCESS="-0.9901"
ERROR_WRONG_PARAM="-0.9902"
ERROR_DATA="-0.9903" # either can not connect /	bad host / bad port

# Handle host and port if non-default
if [ ! -z "$ZBX_REQ_DATA_URL" ]; then
  URL="$ZBX_REQ_DATA_URL"
else
  URL="$NGINX_STATUS_DEFAULT_URL"
fi

# save the nginx stats in a variable for future parsing
NGINX_STATS=$($WGET_BIN -q $URL -O - 2> /dev/null)

# error during retrieve
if [ $? -ne 0 -o -z "$NGINX_STATS" ]; then
  echo $ERROR_DATA
  exit 1
fi

# 
# Extract data from nginx stats
#
case $ZBX_REQ_DATA in
  active_connections)   echo "$NGINX_STATS" | head -1             | cut -f3 -d' ';;
  accepted_connections) echo "$NGINX_STATS" | grep -Ev '[a-zA-Z]' | cut -f2 -d' ';;
  handled_connections)  echo "$NGINX_STATS" | grep -Ev '[a-zA-Z]' | cut -f3 -d' ';;
  handled_requests)     echo "$NGINX_STATS" | grep -Ev '[a-zA-Z]' | cut -f4 -d' ';;
  reading)              echo "$NGINX_STATS" | tail -1             | cut -f2 -d' ';;
  writing)              echo "$NGINX_STATS" | tail -1             | cut -f4 -d' ';;
  waiting)              echo "$NGINX_STATS" | tail -1             | cut -f6 -d' ';;
  *) echo $ERROR_WRONG_PARAM; exit 1;;
esac

exit 0
```

 - Active connections –当前活跃的连接数量  
 - server accepts handled requests — 总共处理了756072922个连接 , 成功创建 756072922次握手, 总共处理了1136799890个请求
 - reading — 读取客户端的连接数.
 - writing — 响应数据到客户端的数量
 - waiting — 开启 keep-alive 的情况下,这个值等于 active – (reading+writing), 意思就是 Nginx 已经处理完正在等候下一次请求指

```text
[root@localhost data]# vi nginx_check.sh  
...
[root@localhost data]# chmod +x nginx_check.sh 
[root@localhost data]# ./nginx_check.sh  handled_requests

[root@localhost data]# vi /usr/local/zabbix/etc/zabbix_agentd.conf
# This is a configuration file for Zabbix agent daemon (Unix)
# To get more information about Zabbix, visit http://www.zabbix.com

############ keys ###############################
UserParameter=nginx[*],/data/nginx_check.sh  "$1"
```

在zabbix server 端进行测试
```text
[root@localhost apache2]# /usr/local/zabbix/bin/zabbix_get  -s  172.16.47.138   -k  nginx[active_connections] 
1

## 失败的情况
[root@localhost apache2]# /usr/local/zabbix/bin/zabbix_get  -s  172.16.47.138   -k  nginx[active_connections]
-0.9903
```


### 添加到zabbix web 配置

主要配置这三项
![Image text](/assets/images/blogs/zabbix-trigger/zd1.png)

这里天些的键值是上面命令 **-k** 的值，我这里先改为 handled_connections

![Image text](/assets/images/blogs/zabbix-trigger/zd2.png)

![Image text](/assets/images/blogs/zabbix-trigger/zd3.png)

报警的条件

![Image text](/assets/images/blogs/zabbix-trigger/zd4.png)

![Image text](/assets/images/blogs/zabbix-trigger/zd5.png)

尝试断开nginx

![Image text](/assets/images/blogs/zabbix-trigger/zd6.png)


## 自定义报警后执行操作

> ps 使用监听项与上面的不同，来自模板 nginx

在触发器监控到主机报警时，能执行相关的操作，如nginx 挂掉后，自动重启 nginx ！

### 准备
![Image text](/assets/images/blogs/zabbix-trigger/t1.png)


### 权限和脚本

需要打开 **sudoer**

创建 `/etc/sudoers.d/zabbix` ，为 zabbix 用户提权
添加

```text
zabbix ALL=(ALL) NOPASSWD: /bin/bash /data/nginx_restart.sh
Defaults:zabbix        !requiretty
```

在 **zabbix_agentd.conf** 中打开
```text 
EnableRemoteCommands = 1
```

添加脚本 **/data/nginx_restart.sh**
```text
#!/bin/sh
/usr/local/nginx/sbin/nginx
```

重启 **zabbix_agentd** !!!

> ps 早 zabbix 端尝试执行 /usr/local/zabbix/bin/zabbix_get  -s  172.16.47.138   -k  system.run["sudo /bin/bash /data/nginx_restart.sh"]，证明zabbix 提权后能执行 sudo(root用户) 命令



意思是 允许使用 root 权限执行 而不需要输入代码

> ps 不要 打开 /etc/sudoer 的 **#includedir /etc/sudoers.d** (最尾一句)  # 和 includedir 是一体的，不是注释
> 不是注释~!
> 不是注释~!
> 不是注释~!

### zabbix web 配置

![Image text](/assets/images/blogs/zabbix-trigger/t2.png)

![Image text](/assets/images/blogs/zabbix-trigger/t3.png)

