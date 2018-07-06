---
title: 'nginx 配置笔记'
date: 2018-07-03
tags:
  - php
  - linux
  - mysql
  - nginx
toc: true
toc_label: "目录"
teaser: /assets/images/teaser/nginx.png
---

## 安装 nginx
[自动化安装nginx shell](https://github.com/jimb55/Exper/blob/master/linux/nginx/nginx.sh)

## 配置支持php

```text
  server {
    listen 80;
    server_name 172.16.47.144;
    access_log /data/wwwlogs/access_nginx.log combined; 
    error_log  /data/wwwlogs/error_nginx.log;
    root /usr/local/nginx/html;
    index index.html index.htm index.php;
    location ~ \.php$ { #所有以 .php 结尾的 链接
      fastcgi_pass 127.0.0.1:9000;# php-fpm的默认端口是9000
      fastcgi_index index.php;   # 默认 index 文件
      include fastcgi.conf;
    }
  }
```

### 支持多模块php 框架

上面的location 只能适用于 .php 结尾的情况
如有些框架的 链接是这样子的
http://172.16.47.144/company.php/info.html  (访问company 模块的 info 操作)
http://172.16.47.144/buy.php/product.html  (访问company 模块的 product 操作)

这时就不是 .php 结尾了,要改变
```text
  server {
    listen 80;
    server_name 172.16.47.144;
    access_log /data/wwwlogs/access_nginx.log combined; 
    error_log  /data/wwwlogs/error_nginx.log;
    root /usr/local/nginx/html;
    index index.html index.htm index.php;
    
    location ~ \.php/ {
      if ($request_uri ~ ^(.+\.php)(/.+?)($|\?)) { }
      fastcgi_pass 127.0.0.1:9000;
      include fastcgi_params;
      fastcgi_param SCRIPT_NAME     $1;
      fastcgi_param PATH_INFO       $2;
      fastcgi_param SCRIPT_FILENAME $document_root$1;
    }
     
    
    location ~ \.php$ {
      include fastcgi.conf;
      fastcgi_pass 127.0.0.1:9000;
      fastcgi_index index.php;
    }
  }
	
```


## 动静分离



```text
worker_processes  1;
events {
    worker_connections  100;
    multi_accept  on; 
}
http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    tcp_nopush     on;
    tcp_nodelay   on;

    gzip on;
    gzip_min_length  1k;
    gzip_buffers     4 16k;
    gzip_http_version 1.1;
    gzip_comp_level  2;
    gzip_types       text/plain application/x-javascript text/css application/xml;
    gzip_vary on;


    client_max_body_size 10m;
    client_body_buffer_size 128k;
    keepalive_timeout  65;

    upstream  linux_web {
      server   intest.com  weight=1  max_fails=2  fail_timeout=30;
    }
    server {
        listen       80;
        server_name  localhost;
        error_log  logs/error.log  info;
        root           html;
        location / {
            root   html;
            index  index.html index.htm;
        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
               
        location ~ [^/]\.php(/|$){
           proxy_set_header Host  $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_pass http://linux_web;
        } 
}
```
