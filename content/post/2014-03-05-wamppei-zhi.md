---
title: "WAMP+ThinkPHP配置"
date: 2014-03-05T18:30:00+08:00
comments: true
categories: [
    "PHP"
]
tags: [
    "ThinkPHP",
    "WAMP",
]
description: "ThinkPHP开发环境WAMP部署，.htaccess，自定义路径，自定义域名"
---

配置ThinkPHP开发环境

1. 开启Rewrite支持.htaccess
2. 自定义工程路径
3. 自定义本地域名访问

<!--more--> 

WAMP安装
http://www.wampserver.com/en/#download-wrapper

```
#编辑wamp/bin/apache/Apache2.x.x/conf/httpd.conf
#开启rewrite，支持.htaccess
#LoadModule rewrite_module modules/mod_rewrite.so去掉注释
LoadModule rewrite_module modules/mod_rewrite.so

#开启httpd-vhosts，自定义域名和工程路径
#Include conf/extra/httpd-vhosts.conf去掉注释
Include conf/extra/httpd-vhosts.conf

```

自定义本地域名和工程路径
```
#编辑wamp/bin/apache/Apache2.x.x/conf/extra/httpd-vhosts.conf
#自定义工程路径
#d:projectpath工程路径
#mylocalhost.com自定义域名
#注意AllowOverride 是All，与wamp主界面不同
#可以定义其他端口
<VirtualHost *:80>  
  DocumentRoot d:projectpath 
  ServerName myhost.com 
  <Directory "d:projectpath"> 
      Options Indexes FollowSymLinks 
      AllowOverride All 
      Order allow,deny 
      Allow from all 
  </Directory> 
</VirtualHost>

#添加wamp主界面，假设wamp安装在D盘
<VirtualHost *:80>  
  DocumentRoot d:wamp/www 
  ServerName wamp.com 
  <Directory "d:wamp/www"> 
      Options Indexes FollowSymLinks 
      AllowOverride None 
      Order allow,deny 
      Allow from all 
  </Directory> 
</VirtualHost>

```

去掉URL中的index.php
```
#.htaccess
<IfModule mod_rewrite.c>
RewriteEngine on
RewriteCond %{SCRIPT_FILENAME} !-f
RewriteCond %{SCRIPT_FILENAME} !-d
RewriteRule ^(.*)$ index.php/$1 [QAS,PT,L]
</IfModule>

```

```
#修改系统hosts
127.0.0.1 	myhost.com
127.0.0.1 	wamp.com

```

重启Apache

查看成果吧
```
myhost.com
wamp.com
```


