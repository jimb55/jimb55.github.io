---
title: 'apache 虚拟主机和Rewrite'
date: 2018-05-19
tags:
  - linux
  - apache
toc: true
toc_label: "目录"
header:
  teaser: /assets/images/teaser/apache.png
---

## 虚拟主机

### 是什么
一台服务器上发布多网站，用不同端口或同端口不同域名访问对应网站目录。

### 有什么用
合理分配资源，例如一台机子只配置一个网站，但这个网站访问自由不到十个人，那就是浪费，配置多几个个网站，使服务器资源合理化运用,不造成浪费.

### 配置实验

虚拟主机类型：
 - 一个IP 不同端口 对应不同的主机（当然可以相同）
 - 一个IP 相同端口 不同域名对应不同主机(一个80端口对应N个域名的例子)
 - 多个IP 对应不同主机 
 
配置实例 ,在
在 `conf/http.conf` 中打开  `httpd-vhosts.conf`
```
# Virtual hosts
Include conf/extra/httpd-vhosts.conf #把这句的#号去掉
```

httpd-vhosts.conf中配置 

```
<VirtualHost *:80>
    ServerAdmin xxx@gmail.com
    DocumentRoot "/data/www/testVhost"
    ServerName jimbtest.com
    ErrorLog "/data/logs/jimbtest_error_log"
    CustomLog "/data/logs/jimbtest_access_log" common

 <Directory "/data/www/testVhost">
  Options -Indexes +FollowSymlinks
  AllowOverride All

    Order Deny,Allow
    Allow from All

  #Require all granted   #注意，apache 2.4 需要用这句打开权限，上面那两句无效
 </Directory>
</VirtualHost>
```

> ps apache 2.4 网页访问无权限 问题

打开`/data/logs/jimbtest_error_log` (以上面的配置例子的log 日志，可查看)
```
[root@localhost ~]# cat  /data/logs/jimbtest_error_log                      

[Mon May 21 10:47:48.410073 2018] [authz_core:error] [pid 6268] [client 172.16.47.1:55760] AH01630: client denied by server configuration: /data/www/testVhost/index.html
[Mon May 21 10:49:54.242726 2018] [authz_core:error] [pid 6269] [client 172.16.47.1:56326] AH01630: client denied by server configuration: /data/www/testVhost/index.html
[Mon May 21 10:49:54.392827 2018] [authz_core:error] [pid 6269] [client 172.16.47.1:56326] AH01630: client denied by server configuration: /data/www/testVhost/index.html
[Mon May 21 10:49:54.555272 2018] [authz_core:error] [pid 6269] [client 172.16.47.1:56326] AH01630: client denied by server configuration: /data/www/testVhost/index.html
[Mon May 21 10:49:54.681105 2018] [authz_core:error] [pid 6269] [client 172.16.47.1:56326] AH01630: client denied by server configuration: /data/www/testVhost/index.html
```

把上面的改成如下即可访问(apache 2.4)

```
<VirtualHost *:80>
    ServerAdmin xxx@gmail.com
    DocumentRoot "/data/www/testVhost"
    ServerName jimbtest.com
    ErrorLog "/data/logs/jimbtest_error_log"
    CustomLog "/data/logs/jimbtest_access_log" common

 <Directory "/data/www/testVhost">
  Options -Indexes +FollowSymlinks
  AllowOverride All
  Require all granted 
 </Directory>
</VirtualHost>
```

## Rewrite规则

### 是什么
支持URL 重写。
例如在浏览器上输入 `baidu.com` 却 打开网页是 地址栏变成 `www.baidu.com`(带https)

### 有什么用?
 - 对搜索引擎优化（**SEO**）友好
 - 隐藏网站URL真实地址，浏览器显示更加美观
 - 网站变更升级，可以基于Rewrite临时重定向到其他页面,例如(打开某些博客论坛时突然飘出 网站维护中...)


### 配置实验

编译安装时需要 加上 `--enable-rewrite`
```
# ./configure --prefix=/usr/local/apache2/ --enable-rewrite --enable-so
```

配置文件`conf/httpd.conf`需要打开
```
LoadModule rewrite_module modules/mod_rewrite.so
```

使用Apache **Rewrite**，除了安装**Rewrite**模块之外，还需在`conf/httpd.conf`或`conf/extra/httpd-vhosts.con`f中的全局配置段或者虚拟主机配置段设置如下指令来开启**Rewrite**功能

```
RewriteEngine on
```

如果 启动 报错 ，检测**mod_rewrite.so**是否正常开启(**上面三点**)
```
[root@localhost ~]# apachectl start                                  
AH00526: Syntax error on line 51 of /usr/local/apache2/conf/extra/httpd-vhosts.conf:
Invalid command 'RewriteEngine', perhaps misspelled or defined by a module not included in the server configuration
```

#### 简单例子
把 jimbtest.com 变换成 www.jimbtest.com
修改`Include conf/extra/httpd-vhosts.conf`

```
<VirtualHost *:80>
    ServerAdmin xxx@gmail.com
    DocumentRoot "/data/www/testVhost"
    ServerName jimbtest.com
    ErrorLog "/data/logs/jimbtest_error_log"
    CustomLog "/data/logs/jimbtest_access_log" common

    RewriteEngine on
    RewriteCond  %{HTTP_HOST}  ^jimbtest.com     [NC]
    RewriteRule ^/(.*)$  http://www.jimbtest.com/$1  [L]

 <Directory "/data/www/testVhost">
  Options -Indexes +FollowSymlinks
  AllowOverride All
  Require all granted
 </Directory>

</VirtualHost>
```

 - RewriteEngine on 启动重写
 - RewriteCond NC  (NC:忽略大小)匹配以jimbtest.com开头的域名,RewriteCond只作用第一个下面的RewriteRule。
 - RewriteRule 执行重写 <u>(.*)</u> 表示任意字符串 $1 表示引用
 - HTTP_HOST 是 <u>**服务器变量**</u> 表示匹配服务器ServerName域名,更多的如HTTP_COOKIE,DOCUMENT_ROOT等
   服务器变量 列表: [http://httpd.apache.org/docs/2.4/expr.html#vars](http://httpd.apache.org/docs/2.4/expr.html#vars)
 - (.*) 是正则匹配,有如下规则
     - . 匹配任何单字符	c.t will match cat, cot, cut, etc.
     - \+ 匹配1到多个字符,Repeats the previous match one or more times	a+ matches a, aa, aaa, etc
     - \* 匹配0到多个字符,Repeats the previous match zero or more times.	a* matches all the same things a+ matches, but will also match an empty string.
     - ? 匹配0到1个字符,Makes the match optional.	colou?r will match color and colour.
     - ^ 以某某开头,Called an anchor, matches the beginning of the string	^a matches a string that begins with a
     - $ 以某某字符结尾 (The other anchor, this matches the end of the string).	a$ matches a string that ends with a.
     - ( ) 分组, 如 (abc){2}只匹配到abcabc，但匹配不到了abc,如 (ab)+ matches ababab - that is, the + applies to the group. For more on backreferences see below.
     - [jm] 匹配字符串jm 	c[uoa]t matches cut, cot or cat.
     - [^jm] 不匹配字符串jm  matches any character not specified	c[^/]t matches cat or c=t but not c/t
     详情 [http://httpd.apache.org/docs/2.4/rewrite/intro.html](http://httpd.apache.org/docs/2.4/rewrite/intro.html)
 - [NC] 这个是 <u>opt Flags</u>,**NC** 决定 RewriteRule 规则的匹配行为可以通过应用程序区分大小写,L(last rule) 匹配当前规则为最后一条匹配规则，停止匹配后续规则,其他如 BNP，B等
   列表 [http://httpd.apache.org/docs/2.4/rewrite/flags.html](http://httpd.apache.org/docs/2.4/rewrite/flags.html)
   
   
综上所述：

![Image text](/assets/images/blogs/apache-httpd-vhost/zsss1.jpg)

[Directive(指令)](http://httpd.apache.org/docs/2.4/mod/mod_rewrite.html#rewriterule)
[服务器变量](http://httpd.apache.org/docs/2.4/expr.html#vars)
[正则](http://httpd.apache.org/docs/2.4/rewrite/intro.html)
[flags](http://httpd.apache.org/docs/2.4/rewrite/flags.html)

### 正则与参数

图片从apache 官网拷贝

![Image text](/assets/images/blogs/apache-httpd-vhost/rs.png)

修改一下
```
    <VirtualHost *:80>
        ServerAdmin xxx@gmail.com
        DocumentRoot "/data/www/testVhost"
        ServerName jimbtest.com
        ErrorLog "/data/logs/jimbtest_error_log"
        CustomLog "/data/logs/jimbtest_access_log" common
    
        RewriteCond %{DOCUMENT_ROOT}/$1 !-f
        RewriteCond %{HTTP_HOST}  ^(jimbtest.com)$    [NC]
        RewriteRule ^/?([a-z]+)/(.*)$  /admin.foo?page=$1&id=$2&host=%1  [PT]
    
     <Directory "/data/www/testVhost">
      Options -Indexes +FollowSymlinks
      AllowOverride All
      Require all granted
     </Directory>
    </VirtualHost>
```
> ps : RewriteCond 指令定义了一个规则条件。一个或多个RewriteCond可以在一个RewriteRule指令之前。ps:不作用于第二个RewriteRule.
 
 - 1 访问 `http://jimbtest.com/a/t.html`  执行 第一句 `RewriteCond`
 - 2 `$1` 是 `a`, `!-f` 表示不存在这份文件才走下去，若 `DOCUMENT_ROOT` 中存在a文件，就是说我这里的 `/data/www/testVhost/a`是文件的话，匹配失败
 - 3 `HTTP_HOST` 是域名， NC是不分大小写情况下匹配
 - 4 `%1` 是 `jimbtest.com` ,`$1`是 `a`, `$2` 是 `t.html`
 - 5 `PT` 是使 `RewriteRule`的结果(就是`/admin.foo?page=a&id=t.html&host=jimbtest.com`) 作为URL映射传回 (to be passed back through URL mapping 谷歌翻译的...),就是结果能被 `Alias, Redirect, or ScriptAlias`等**指令**处理处理

#### `PT` 测试:
如下代码，若访问 `http://jimbtest.com/pics/t1.jpg` 能正确访问到 `/data/file/image/t1.jpg`
但若是 没有 `PT` ，即 `RewriteRule` 匹配成功，作为文件访问,即是 `http://base.jimbtest.com/image/$1.png`,
并不会再执行 `ALias` 映射，而变成访问  `/data/file/image/$1.png`

下面的能正确访问到 

```
Alias "/image/" "/data/file/image/"
RewriteRule "/pics/(.+)\.jpg$" "/image/$1.png"  [PT]
```