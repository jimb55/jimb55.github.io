---
title: 'linux shell'
date: 2018-06-21
tags:
  - linux
toc: true
toc_label: "目录"
header:
  teaser: /assets/images/teaser/shell.jpg
---

## 流程控制 [] 和 [[]]

&& ||

```text
if [[ $a != 1 && $a != 2 ]]  =  if [ $a -ne 1] && [ $a != 2 ]  =   if [ $a -ne 1 -a $a != 2 ]
```

注意 && || 能出现在 [[]] 中 ，[] 内 只能写成 -a等

支持字符串的模式匹配 
```text
#!/bin/bash

if [[ $USER == r* ]]
then 
	echo "Hello $USER"
else
	echo "Sorry, I do not know you"
fi
```

root 用户会输出 Hello root ,变成 [] 后 ，输出 **Sorry, I do not know you** 因为不识别 ``r*``

## 分割符 IFS

IFS=‘\n’  //将字符n作为IFS的换行符。
IFS=$"\n" //这里\n确实通过$转化为了换行符，但仅当被解释时（或被执行时）才被转化为换行符。
IFS=$'\n' //这才是真正的换行符。

```text
#!/bin/bash
#changing the IFS value
IFS.OLD=$IFS
IFS=$'\n'
for entry in `cat /etc/passwd`
do	
	echo "Values in $entry -"
	IFS=:
	for value in $entry
	do
		echo " $value"
	done
done

```

```text
#!/bin/bash
oldIFS=$IFS #IFS是文件内部分隔符
IFS=":"     #设置分隔符为:
while  read username var #var变量不可少
do
    echo "用户名:$username"
done < /etc/passwd 
IFS=$oldIFS
```
>熟悉/etc/passwd文件结构的朋友都知道这个文件的每一行包含了一个用户的大量信息（用户名只是第一项）。 
 这里我们实际只输出了用户名。但是注意while的read后面除变量username外还有个var，尽管我们并不输出这个变量的值。 
 但它却必不可少，如果我们写成while read username那么username的值等于passwd文件这一整行的内容（IFS=”:”也就不起作用了）


## 重定向
&> 等如 2>&1 , > 等如 1> ,那是缩写

```text
nginx -V 2>123
```

```text
command | while read line
do
    …
done

while read line
do
       …
done < file
```
> read -d: 定义分割符为 ：