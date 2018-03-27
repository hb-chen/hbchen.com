---
title: "MySQL:常用命令行"
date: 2013-10-30T08:58:00+08:00
comments: true
categories: [
	"DB",
]
tags: [
	"MySQL",
]
description: "MySQL常用的命令行操作"
---

#### 登入
```
mysql -h192.168.1.110  -uroot -ppassword
```
 
#### 登出
```
quit/exit
```

<!--more-->  
 
#### 查看数据库
```
show databases;
```
 
#### 用户权限
```
#添加
grant select on db.table to 'user'@'host';
grant select,update on *.* to 'test'@'%';
#撤销
revoke all on *.* from 'test'@'%';
#查看
show grants;
show grants for user@localhost
 
#删除用户
delete from mysql.user where user='' and host='';

#设置密码
update mysql.user set password=PASSWORD('123456') where user='root';
 
#配置远程连接
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'password' WITH GRANT OPTION; 
 ```

#### 导出  
mysqldump -u　用户名 -p 数据库名 > 数据库名.sql 
```
mysqldump -u　root -p dbname > filepath.sql  
#To export tofile(data only)
mysqldump -u [user]-p[pass]--no-create-db --no-create-info mydb > mydb.sql
#To export tofile(structure only)
mysqldump -u [user]-p[pass]--no-data mydb > mydb.sql
#To import todatabase
mysql -u [user]-p[pass] mydb < mydb.sql
```

#### 导入
```
mysql> use dbname             #切到要导入的数据库
mysql> source filepath.sql
```
 
#### 变量查看/修改
```
show variables like '%slow%';
set global slow_query_log= 'ON';

show status like 'Qca%';
 
#查看SQL中select条数
show status like 'Com_sel%'; #显示的是一次会话的值！
show global status like "Com_select";
```
 
#### 找回密码？
```
1、kill -TERM mysqld
2、conf中加入skip-grant-tables
3、service mysqld restart
4、mysql -uroot -p
5、修改密码
6、改回原配置并重启
```

#### [Binlog](http://dev.mysql.com/doc/refman/5.7/en/mysqlbinlog.html)
[删除mysql-bin日志](http://dev.mysql.com/doc/refman/5.7/en/purge-binary-logs.html)
```
PURGE BINARY LOGS TO 'mysql-bin.010';
PURGE BINARY LOGS BEFORE '2008-04-02 22:46:26';
```
 
#### 清空表并使自增归0
```
TRUNCATE TABLE tablename
```
 
#### Mac无法登陆
```
Can't connect to MySQL server on '127.0.0.1' (61)
StevenMacBookAir:~ Hobo$ sudo su -
StevenMacBookAir:~ root# nohup /usr/local/mysql/bin/mysqld_safe &
[1] 464
StevenMacBookAir:~ root# appending output to nohup.out
exit
logout
StevenMacBookAir:~ Hobo$ 
```

