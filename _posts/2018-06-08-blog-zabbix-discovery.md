---
title: 'zabbix 批量监控'
date: 2018-06-08
tags:
  - linux
  - zabbix
toc: true
toc_label: "目录"
header:
  teaser: /assets/images/teaser/zabbix.png
---

## 自动发现(discovery)

自动发现是 `zabbix server` 的 `Discover` 程序扫描IP 地址，将符合条件的IP机子放到host配置中

### 配置操作

![Image text](/assets/images/blogs/zabbix_discovery/zd1.png)

![Image text](/assets/images/blogs/zabbix_discovery/zd2.png)

如果配置后没有找到机子
查看 check 方式（例如我这里是 ssh 22 端口） 对应的端口是否开启
```text
[root@localhost local]# firewall-cmd --zone=public --add-port=22/tcp --permanent
success
[root@localhost local]# firewall-cmd --reload
success
[root@localhost local]# firewall-cmd --list-ports
10050/tcp 22/tcp
```

![Image text](/assets/images/blogs/zabbix_discovery/zd3.png)

![Image text](/assets/images/blogs/zabbix_discovery/zd4.png)

![Image text](/assets/images/blogs/zabbix_discovery/zd5.png)

最后在 config 上查看生成 host 配置文件

![Image text](/assets/images/blogs/zabbix_discovery/zd6.png)

### 部署监听agent
```text
[root@localhost local]# scp -r zabbix/ root@172.16.47.137:/usr/local/
root@172.16.47.137's password: 
zabbix_agentd             100% 1379KB  44.8MB/s   00:00    
zabbix_server             100% 4498KB  34.2MB/s   00:00    
zabbix_server.conf.bak    100%   14KB   3.3MB/s   00:00    
zabbix_server.conf        100%   14KB   7.3MB/s   00:00    
zabbix_agentd.conf        100%   10KB   5.4MB/s   00:00    
zabbix_get                100%  394KB  30.7MB/s   00:00    
zabbix_sender             100%  468KB  17.0MB/s   00:00    
zabbix_get.1              100% 3509   396.2KB/s   00:00    
zabbix_sender.1           100%   12KB   8.9MB/s   00:00    
zabbix_agentd.8           100% 3048   335.8KB/s   00:00    
zabbix_server.8           100% 2045   600.8KB/s   00:00    
[root@localhost local]# scp -r /etc/init.d/zabbix_agentd root@172.16.47.137:/etc/init.d 
root@172.16.47.137's password: 
zabbix_agentd             100% 1519     1.3MB/s   00:00   
```


修改该对应配置文件
```text
...
Server=172.16.47.142
ServerActive=172.16.47.142
Hostname = 172.16.47.143
...
```

在部署的机子启动 agent
```text
[root@testvhost zabbix]# /etc/init.d/zabbix_agentd restart
Zabbix agent started.
```

![Image text](/assets/images/blogs/zabbix_discovery/zd7.png)


## 自动注册(Auto registration)

自动验证是 `agent` 端在启动的时候 发送请求到  `zabbix server` 端，并按相应的规则添加  `zabbix server` 配置，前提是`agent` 端 必须要配置好 `zabbix_agent` ,同时配置文件 `zabbix_agentd.conf` 指向`zabbix server`IP;

配置大致如上，但需要更改 ACTION 的配置


![Image text](/assets/images/blogs/zabbix_discovery/zd1.png)

![Image text](/assets/images/blogs/zabbix_discovery/zd2.png)

如果配置后没有找到机子
查看 check 方式（例如我这里是 ssh 22 端口） 对应的端口是否开启
```text
[root@localhost local]# firewall-cmd --zone=public --add-port=22/tcp --permanent
success
[root@localhost local]# firewall-cmd --reload
success
[root@localhost local]# firewall-cmd --list-ports
10050/tcp 22/tcp
```

![Image text](/assets/images/blogs/zabbix_discovery/zd8.png)

![Image text](/assets/images/blogs/zabbix_discovery/zd9.png)

![Image text](/assets/images/blogs/zabbix_discovery/zd10.png)

![Image text](/assets/images/blogs/zabbix_discovery/zd11.png)
