---
title: 'php 运行方式'
date: 2018-07-04
tags:
  - php
  - linux
  - nginx
toc: true
toc_label: "目录"
teaser: /assets/images/teaser/php1.jpg
---

## php 运行方式

前置知识
[php 与 cgi](https://www.cnblogs.com/f-ck-need-u/p/7627035.html)
[php 与 opcode](https://tech.youzan.com/understanding-opcode-optimization-in-php/)

### CGI
使用CGI模式时，当动态请求到达，httpd临时启动一个cgi解释器，并通过cgi协议转发要运行的内容。当cgi脚本运行结束后，将结果返回给httpd，然后cgi解释器进程自我销毁。当多个动态请求到达时，将先后启动多个cgi解释器。因此，这种方法效率极低。


#### python 和 php cgi配置

配置文件中->
```text
LoadModule cgid_module modules/mod_cgid.so 

AddHandler cgi-script .cgi .py .phtml

ScriptAlias /cgi-bin/ "/tmp/"   # 确保有权限的执行目录

#目录权限配置
<Directory "/tmp">
  AllowOverride None
  Options +ExecCGI
  AllowOverride All
  Require all granted
</Directory>

```
模块是这样子的
LoadModule cgid_module modules/mod_cgid.so  #//当使用内置模块prefork.c 时动态加载cgi_module
LoadModule cgi_module modules/mod_cgi.so   #当使用内置模块worker.c 时动态加载cgid_module


在 /tmp 下准备两份执行脚本
index.py
```
#!/usr/bin/env python
# -*- coding: UTF-8 -*-

print "Content-type:text/html"
print
print '<html>'
print '<head>'
print '<title>Hello</title>'
print '</head>'
print '<body>'
print '<h2>this is py cgi </h2>'
print '</body>'
print '</html>'
```

index.phtml
```
#!/usr/local/php7/bin/php
<?php
print "Content-type:text/html\n\n";  

echo "this is phtml";
```

且确认都有执行权限
```text
chmod 755 /tmp/index.py
chmod 755 /tmp/index.phtml
```

重启apache 便可访问!

>ps 若遇到以下报错

报错 
End of script output before headers: index.phtml
或
malformed header from script 'index.phtml': Bad header: phpinfo()

在 php 脚本中加上以下内容
```text
<?php
print "Content-type:text/html\n\n"; //因为没有输出http 报文头的Content-type,web server 不识别
```

#### php-cgi 处理 .php 文件
同样，打开CGI模块
```text
LoadModule cgid_module modules/mod_cgid.so 

#找到mime
<IfModule mime_module>
    AddType application/x-httpd-php .php
    #表示 吧.php 文件看做成  application/x-httpd-php 这用应用数据
    Action  application/x-httpd-php '/cgi-bin/php-cgi'
    # 把application/x-httpd-php这种内容交给  /usr/loacl/apache2/cgi-bin/php-cgi 处理(理论上)
    # 但是实际 确实 /tmp/htdocs/php-cgi ，因为下面配置了ScriptAlias ,把 /cgi-bin/  映射到 /tmp/htdocs/
</IfModule>

# 配置脚本目录 注意apache 执行用户需要目录的读写权限
<IfModule alias_module>
    ScriptAlias /cgi-bin/ "/tmp/htdocs/"
</IfModule>

#配置目录 确保脚本放在一个个能被apache 当前执行 用户访问的 目录
DocumentRoot "/tmp/htdocs"
<Directory "/tmp/htdocs">
    Options Indexes FollowSymLinks ExecCGI
    AllowOverride None
    Require all granted
</Directory>
```
> ps 注意此处的 /cgi-bin/ ，这里 的 / 是指 apache 的 根目录，你要把 php-cgi 拷贝到 /usr/loacl/apache2/cgi-bin 下，或者软连

> ps 注意配置Directory 和 ScriptAlias时 ， apache 用户 要有读写权限的 目录


### FASTCGI

nginx 使用 php-fpm 就是例子
nginx 通过 fastcgi 协议 发送内容给 php-fpm
php-fpm 是 php-cgi 的管理器 
php-cgi 实现了 fastcgi 协议，并执行内容

php-cgi 运行过程
![Image text](https://images2017.cnblogs.com/blog/733013/201710/733013-20171004193753208-1919724771.png)


### php_mod
就是以web server 模块的方式运行 ,如apache 的 php7_module : **modules/libphp7.so**       

### CLI
控制台命令的方式运行

### ISAPI