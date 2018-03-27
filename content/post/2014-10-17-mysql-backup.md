---
title: "MySQL Backup"
date: 2014-10-17T14:04:00+08:00
comments: true
categories: [
	"DB",
]
keywords: [
	"MySQL",
	"备份",
]
---

MySQL备份脚本，支持mysqldump,mysqlhotcopy,tar三种方式，+定时任务自动备份。

Gist
https://gist.github.com/Hobo86/effd4b45b50f576bf4d1

<!--more-->

```
#脚本属性设为可执行
$ chmod +x mysql_backup.sh
 
#编辑定时任务
$ vi /etc/crontab
 
如：每天03:01执行备份脚本
01 3 * * * root /usr/sbin/mysql_backup.sh
 
#重启定时任务
$ /etc/rc.d/init.d/crond restart
```

Gist
https://gist.github.com/Hobo86/29b27d361a4c59545348

```
#mysql_backup.sh
#!/bin/bash
 
DBName=db_name
 
DBUser=root
 
DBPasswd=123456
 
BackupPath=/mnt/backup/
 
LogFile=/mnt/backup/db_name.log
 
DBPath=/mnt/mysql/
 
BackupMethod=mysqldump
 
#BackupMethod=mysqlhotcopy
 
#BackupMethod=tar
 
 
NewFile="$BackupPath"db_name_$(date +%y%m%d).tgz
 
DumpFile="$BackupPath"db_name_$(date +%y%m%d)
 
OldFile="$BackupPath"db_name_$(date +%y%m%d --date='5 days ago').tgz
 
#SettingEnd
 
echo "-------------------------------------------">>$LogFile
echo $(date +%Y-%m-%d%t%H:%M:%S)>>$LogFile
 
echo "--------------------------">>$LogFile
 
#DeleteOldFile
if [ -f $OldFile ]
	then
		rm -f $OldFile>>$LogFile 2>&1
		echo "[$OldFile]DeleteOldFileSuccess!">>$LogFile
	else
		echo "[$OldFile]NoOldBackupFile!">>$LogFile
fi
 
if [ -f $NewFile ] 
	then
		echo "[$NewFile]TheBackupFileisexists,Can'tBackup!">>$LogFile
	else
		case $BackupMethod in
		mysqldump)
 			if [ -z $DBPasswd ] 
				then
					mysqldump -u$DBUser --opt $DBName>$DumpFile
				else
					mysqldump -u$DBUser -p$DBPasswd --opt $DBName>$DumpFile
			fi
 
			tar czvf $NewFile $DumpFile>>$LogFile 2>&1
			echo "[$NewFile]BackupSuccess!">>$LogFile
 			rm -rf $DumpFile
		;;
 
		mysqlhotcopy)
 			rm -rf $DumpFile
 			mkdir $DumpFile
 
			if [ -z $DBPasswd ] 
				then
					mysqlhotcopy -u$DBUser $DBName $DumpFile>>$LogFile 2>&1
				else
					mysqlhotcopy -u$DBUser -p$DBPasswd $DBName $DumpFile>>$LogFile2>&1
			fi
 
			tar czvf $NewFile $DumpFile>>$LogFile 2>&1
			echo "[$NewFile]BackupSuccess!">>$LogFile
			rm -rf $DumpFile
		;;
 
		*)
			/etc/init.d/mysqldstop>/dev/null2>&1
			tar czvf $NewFile $DBPath$DBName>>$LogFile 2>&1
			/etc/init.d/mysqldstart>/dev/null2>&1
			echo "[$NewFile]BackupSuccess!">>$LogFile
		;;
 
		esac
fi
echo "-------------------------------------------">>$LogFile
```