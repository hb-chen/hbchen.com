---
title: "日志统计分析-Shell/Goaccess"
date: 2013-11-07T18:36:00+08:00
comments: true
categories: [
	"系统",
]
tags: [
	"日志",
	"Goaccess",
	"Shell",
]
description: "使用Shell和Goaccess简单统计分析日志文件"
---
对Nginx或其他日志进行简单的统计分析
 
Shell
对某一列进行统计，可以分析Status Code,URL等
```
#cat access.log | awk '{print $9}'|sort|uniq -c | sort -r -n > stat.log
或
#cat access.log |grep "200" | awk '{print $7}'|sort|uniq -c | sort -r -n > stat.log
#vi stat.log
```

<!--more-->  
 
指定String统计
```
#cat access.log|grep "200"|wc -l 
#cat access.log|grep "www.localhost.com"|wc -l
```

Goaccess工具
http://goaccess.io/

```
#goaccess -f access.log
#goaccess -f access.log -a -s -b
```
 
Goaccess分析压缩日志
```
#zcat access.log-20130123.gz | goaccess
```
