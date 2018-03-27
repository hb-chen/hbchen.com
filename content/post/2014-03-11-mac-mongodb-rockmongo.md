---
title: "Mac安装配置MongoDB+RockMongo"
date: 2014-03-11T18:00:00+08:00
comments: true
categories: [
	"DB",
]
tags: [
	"MongoDB",
	"RockMongo",
]
---

MongoDB安装
</br>使用brew安装很方便
</br>http://docs.mongodb.org/manual/tutorial/install-mongodb-on-os-x/

安装完成后可以选择修改配置文件
```
#mongod.conf
#dbpath,logpath,bind_ip
vi /usr/local/etc/mongod.conf

```

<!--more--> 

启动配置
```
#为了方便使用配置.bash_profile
vi ~/.bash_profile

#添加以下内容
export PATH=$PATH:/usr/local/opt/mongodb/bin
alias mongodb_start='sudo launchctl load -w /usr/local/Cellar/mongodb/2.4.9/homebrew.mxcl.mongodb.plist'
alias mongodb_stop='sudo launchctl unload -w /usr/local/Cellar/mongodb/2.4.9/homebrew.mxcl.mongodb.plist'
alias mongodb_restart='mongodb_stop; mongodb_start;'

#这样直接使用mongodb_start,mongodb_stop,mongodb_restart很方便

#启动
mongodb_start

#配置用户名密码
mongo
db show
use test
db.addUser("root", "123456")

#重启
mongodb_restart
```

管理工具RockMongo，下载后根据自己的PHP环境配置
</br>http://rockmongo.com/downloads

安装php-mongo
```
#我用的php54，记下安装后的路径
brew php54-mongo

#配置php.ini
#添加或者修改extension="mongo.so"
extension="/usr/local/Cellar/php54-mongo/1.4.5/mongo.so"

#启动/重启Php环境
```





