---
title: "SVN:使用"
date: 2013-11-07T18:15:00+08:00
comments: true
categories: [
	"VCS",
]
tags: [
	"SVN",
	"配置",
	"部署",
]
description: "Centos 安装SVN，配置，自动部署等"
---
#### 1.系统
Centos 6.4 

#### 2.安装svn
```
#yum install subversion
```

<!--more-->  

#### 3.建立版本库
```
#mkdir /mnt/svndata
```
启动服务
```
#svnserve -d -r /mnt/svndata
#svnadmin create /mnt/svndata/svn
```
#### 4.修改配置
```
#cd /mnt/svndata/svn/conf
#vi svnserve.conf
anon-access=none
auth-access=write
password-db=passwd

#vi passwd
[users]
hobo=123456
```

#### 5.实现SVN与WEB同步

1)checkout一个test项目
```
#svn co svn://localhost/svn /mnt/www/webroot/test
```
2)修改权限为WEB用户
```
#chown -R apache:apache /mnt/www/webroot/test
```
3)建立同步脚本
```
#cd /mnt/svndata/svn/hooks/
#cp post-commit.tmpl post-commit
```
4)编辑post-commit，添加同步脚本
```
#vi post-commit
export LANG=en_US.UTF-8
SVN=/usr/bin/svn
WEB_TEST=/mnt/www/webroot/test
$SVN update $WEB_TEST –username hobo –password 123456
chown -R apache:apache $WEB
```
5)增加脚本执行权限
```
#chmod +x post-commit
```


Mac下Versions中.a包无法上传问题
```
cd filepath
svn add libMobClickLibrary.a 
```
 
删除版本控制 / 删除多级.svn文件
```
#find . -type d -name ".svn"|xargs rm -rf;
```


