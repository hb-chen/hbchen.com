---
title: "Centos VSFTP配置"
date: 2014-10-17T16:18:00+08:00
comments: true
categories: [
	"系统",
]
---

安装
```shell
yum install vsftp
```

<!--more-->

配置
```shell
#/etc/vsftpd/vsftpd.conf
#关闭匿名登录
#anonymous_enable=NO

user_list中的说明是userlist_deny
#userlist_enable=NO

FTP root
#local_root=/mnt/ftp

```

添加登录用户
```shell
添加用户
$ useradd Hobo
设置密码
$ passwd Hobo
$加入user_list
$ echo Hobo >> /etc/vsftpd/user_list
```

重启
```
$service vsftpd restart
```