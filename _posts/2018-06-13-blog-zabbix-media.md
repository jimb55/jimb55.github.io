---
title: 'zabbix 报警通知方式'
date: 2018-06-13
tags:
  - linux
  - zabbix
toc: true
toc_label: "目录"
header:
  teaser: /assets/images/teaser/zabbix.png
---

## 发送邮件
配置 发生 问题时 发送邮件到管理员的邮箱

![Image text](/assets/images/blogs/zabbix-media/om1.png)

![Image text](/assets/images/blogs/zabbix-media/om2.png)

![Image text](/assets/images/blogs/zabbix-media/om3.png)

![Image text](/assets/images/blogs/zabbix-media/om4.png)

![Image text](/assets/images/blogs/zabbix-media/om5.png)

![Image text](/assets/images/blogs/zabbix-media/om6.png)

![Image text](/assets/images/blogs/zabbix-media/om7.png)

![Image text](/assets/images/blogs/zabbix-media/om8.png)

![Image text](/assets/images/blogs/zabbix-media/om9.png)

查看监听的项目中，有、boot，尝试提升 监听机子的 swap 分区来测试报警
```text
[root@localhost boot]# pwd
/boot
[root@localhost boot]# dd if=/dev/zero of=test.img bs=1M count=1000
dd: 写入"test.img" 出错: 设备上没有空间
记录了890+0 的读入
记录了889+0 的写出
932683776字节(933 MB)已复制，22.2463 秒，41.9 MB/秒
[root@localhost boot]# df -h
文件系统                 容量  已用  可用 已用% 挂载点
/dev/mapper/centos-root   17G  2.4G   15G   14% /
devtmpfs                 737M     0  737M    0% /dev
tmpfs                    748M     0  748M    0% /dev/shm
tmpfs                    748M  8.6M  739M    2% /run
tmpfs                    748M     0  748M    0% /sys/fs/cgroup
/dev/sda1               1014M 1014M   20K  100% /boot
tmpfs                    150M     0  150M    0% /run/user/0
```

![Image text](/assets/images/blogs/zabbix-media/om10.png)

尝试修复
```text
[root@localhost boot]# rm -rf test.img 
```

修复后收到修复邮件

## 执行指定脚本

配置流程如上， `MEDIA TYPE` 修改如下

![Image text](/assets/images/blogs/zabbix-media/om11.png)

1 是 脚本的名字 ，保存的位置 默认 是 **/usr/local/share/zabbix/alertscripts**,是配置文件 中 **AlertScriptsPath**的值
更多配置参考: [Zabbix server configuration](https://www.zabbix.com/documentation/3.2/manual/appendix/config/zabbix_server)

2 执行脚本时带入的参数 
参数列表: [Macros supported by location](https://www.zabbix.com/documentation/3.2/manual/appendix/macros/supported_by_location)


## 通过微信 企业号 发送


## 通过微信公众号报警
实现思路，通过脚本（php/java/python）实现公众号api，推送报错内容









