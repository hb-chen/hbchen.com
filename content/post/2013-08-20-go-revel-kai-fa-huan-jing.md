---
title: "Go-Revel:开发环境"
date: 2013-08-20T16:38:00+08:00
categories: [
	"Go",
]
tags: [
	"Go",
	"Revel",
]
description: "Go以及Revel框架的搭建"
---

环境安装都直接看官方文档最靠谱，经他人加工后的经常容易误导，后面也会有一些安装体会分享

Go<br>
英文http://golang.org/doc/install#osx	（墙）<br>
中文http://zh.golanger.com/doc/install#osx

Revel<br>
http://robfig.github.io/revel/tutorial/gettingstarted.html

<!--more-->  

Revel配置过程中GOPATH和Revel command的配置可以直接编辑.bash_profile，个人感觉这样更简单直接
```shell
#编辑.bash_profile
$ vi ~/.bash_profile
```

```shell
#添加两行
export GOPATH=gocode_path 			#gocode_path自己的工作目录
export PATH=$PATH:$GOPATH/bin		#PATH可能原来也有其他的配置，继续追加就行，":"分割

#编辑完成，重启终端
```

开发工具我还是喜欢Sublime Text 2，LiteIDE也试过，感觉一般<br>
http://www.sublimetext.com/

再推荐个Mac小工具Go2Shell，安装后添加到Finder的工具栏上，直接在当前目录下启动终端，没用过的同学可以装下，很方便<br>
https://itunes.apple.com/cn/app/go2shell/id445770608?mt=12
