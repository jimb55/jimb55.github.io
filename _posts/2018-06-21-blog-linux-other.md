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
