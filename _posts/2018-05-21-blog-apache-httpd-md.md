---
title: 'apache 模块与指令（有空练习）'
date: 2018-05-21
tags:
  - linux
  - apache
toc: true
toc_label: "目录"
header:
  teaser: /assets/images/teaser/apache.png
---

模块展开为指令列表

[官方 指令索引](http://httpd.apache.org/docs/2.4/mod/directives.html)

## core
Core Apache HTTP Server features that are always available
apache 的核心模块

## mod_headers
自定义 http的  request 和 response headers

### Header
```text
<VirtualHost *:80>
   Header set Set-Cookie 'time=jjj'   
   # 设置cookie
<VirtualHost *:80>
```
[header](http://httpd.apache.org/docs/2.4/mod/mod_headers.html#header)

### AcceptFilter 



## log_config AND logic

logic 是自定义的

```text
<VirtualHost *:80>
    ServerAdmin xxx@gmail.com
    DocumentRoot "/var/www/html"
    ServerName intest.com
    ErrorLog "/data/logs/jimbtest_zabbix_error_log"
    #CustomLog "/data/logs/jimbtest_zabbix_access_log" common

     <IfModule log_config_module>
      LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-agent}i\" %I %O" logic
      CustomLog "/var/logs/httpd/map_log" logic
     </IfModule>

 <Directory "/var/www/html">
  Options -Indexes +FollowSymlinks
  AllowOverride All
  Require all granted
 </Directory>
</VirtualHost>
```
