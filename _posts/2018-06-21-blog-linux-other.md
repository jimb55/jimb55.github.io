---
title: 'linux 知识'
date: 2018-06-21
tags:
  - linux
toc: true
toc_label: "目录"
header:
  teaser: /assets/images/teaser/linux.png
---

## 开机启动 rc.local

简单点就是开机时执行的**shell**脚本

```text
[root@localhost ~]# cat /etc/rc.d/rc.local 
#!/bin/bash
# THIS FILE IS ADDED FOR COMPATIBILITY PURPOSES
#
# It is highly advisable to create own systemd services or udev rules
# to run scripts during boot instead of using this file.
#
# In contrast to previous versions due to parallel execution during boot
# this script will NOT be run after all other services.
#
# Please note that you must run 'chmod +x /etc/rc.d/rc.local' to ensure
# that this script will be executed during boot.

touch /var/lock/subsys/local

echo "kai ji qi dong " > /tmp/rc.local
```

注意需要执行 **Please note that you must run 'chmod +x /etc/rc.d/rc.local' to ensure** 来启动开机启动脚本shell

## VMware centos 共享文件夹
[VMware centos 共享文件夹](https://blog.csdn.net/qq_34129637/article/details/78949662)

```text
/usr/bin/vmhgfs-fuse .host:/ /mnt/hgfs/ -o subtype=vmhgfs-fuse,allow_other
```


## ab 测试出现 Too many open files

Benchmarking 172.16.47.144 (be patient)
socket: Too many open files (24)

```text
[root@intest ng]# ab -n 2000 -c 2000 http://172.16.47.144/
This is ApacheBench, Version 2.3 <$Revision: 1430300 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking 172.16.47.144 (be patient)
socket: Too many open files (24)
```

把 ulimit -n 调大

```text
[root@intest ng]# ulimit -a
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 5890
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files                      (-n) 1024 **这个数目**
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
max user processes              (-u) 5890
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited

[root@intest ng]# ulimit -n 204800
```

> ps 注意是 测试机（就是使用ab 发送请求的测试使用机子） 和 被测试机子（就是安装 nginx、apache 等的机子） 都需要 **open files** 设置 就是 **ulimit -n 204800**

可以使用 以下命令监听 进程打开的文件数 **14894** 是进程 **pid**
```
watch -n1 -d "lsof  -p 14894| wc -l"
```

## 注意软件的删除
```text
[root@localhost cgi-bin]# ls -al
总用量 16
drwxr-xr-x.  2 root root   93 7月   5 15:11 .
drwxr-xr-x. 15 root root  175 7月   5 13:40 ..
lrwxrwxrwx.  1 root root   16 7月   5 15:11 php-cgi -> /usr/bin/php-cgi
-rw-r--r--.  1 root   40  820 12月 18 2012 printenv
-rw-r--r--.  1 root   40 1074 12月 18 2012 printenv.vbs
-rw-r--r--.  1 root   40 1133 12月 18 2012 printenv.wsf
-rw-r--r--.  1 root   40 1261 12月 18 2012 test-cgi
[root@localhost cgi-bin]# rm -rf php-cgi  # 注意，这是是等于删除/usr/bin/php-cgi （源文件） 十分危险

[root@localhost cgi-bin]# rm -rf ./php-cgi # 这个才是把软连删了 而不是源文件
```