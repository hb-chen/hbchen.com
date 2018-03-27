---
title: "Go-Revel:Gorp连接MySQL"
date: 2013-08-24T09:22:00+08:00
categories: [
	"Go",
	"DB",
]
tags: [
	"Go",
	"Revel",
	"数据库",
	"MySQL",
]
description: "Revel使用gorp连接MySQL数据库"
comments: true
---

Revel连接数据库可以参考官方的Booking示例

除了Revel需要Gorp和MySQL驱动两个库
```shell
#Gorp
go get github.com/coopernurse/gorp

#MySQL驱动
go get github.com/go-sql-driver/mysql  #也可以用其他的驱动库
```

创建一个新的Revel工程app_name

<!--more-->  

配置conf/app.conf
```
# DB username:@tcp(yourhost:port)/dbname
db.import = github.com/Go-SQL-Driver/mysql
db.driver = mysql
db.spec   = root:@tcp(localhost:3306)/db_name
```

创建models/user.go，先只定义几个测试字段
```go
package models

type User struct {
	UserId							int
	Name               	string
}
```

创建controllers/init.go
```go
package controllers

import "github.com/robfig/revel"

func init() {
	revel.OnAppStart(Init)
	revel.InterceptMethod((*GorpController).Begin, revel.BEFORE)
	revel.InterceptMethod((*GorpController).Commit, revel.AFTER)
	revel.InterceptMethod((*GorpController).Rollback, revel.FINALLY)
}
```

创建controllers/gorp.go
```go
package controllers

import (
	"database/sql"
	"github.com/coopernurse/gorp"
	_ "github.com/Go-SQL-Driver/mysql"
	r "github.com/robfig/revel"
	"github.com/robfig/revel/modules/db/app"
	"app_name/app/models"
)

var (
	Dbm *gorp.DbMap
)

func Init() {
	db.Init()
	//MySQL InnoDB UTF8
	Dbm = &gorp.DbMap{Db: db.Db, Dialect: gorp.MySQLDialect{"InnoDB", "UTF8"}}

	setColumnSizes := func(t *gorp.TableMap, colSizes map[string]int) {
		for col, size := range colSizes {
			t.ColMap(col).MaxSize = size
		}
	}

	//创建一个User测试表
	t := Dbm.AddTable(models.User{}).SetKeys(true, "UserId")
	setColumnSizes(t, map[string]int{
		"Name": 20,
	})

	Dbm.TraceOn("[gorp]", r.INFO)
	Dbm.CreateTables()

	//插入一组测试数据
	demoUser := &models.User{0, "Hobo"}
	if err := Dbm.Insert(demoUser); err != nil {
		panic(err)
	}

}

type GorpController struct {
	*r.Controller
	Txn *gorp.Transaction
}

func (c *GorpController) Begin() r.Result {
	txn, err := Dbm.Begin()
	if err != nil {
		panic(err)
	}
	c.Txn = txn
	return nil
}

func (c *GorpController) Commit() r.Result {
	if c.Txn == nil {
		return nil
	}
	if err := c.Txn.Commit(); err != nil && err != sql.ErrTxDone {
		panic(err)
	}
	c.Txn = nil
	return nil
}

func (c *GorpController) Rollback() r.Result {
	if c.Txn == nil {
		return nil
	}
	if err := c.Txn.Rollback(); err != nil && err != sql.ErrTxDone {
		panic(err)
	}
	c.Txn = nil
	return nil
}

```

修改controllers/app.go
```go
#import添加
"app_name/app/models"

#包含GorpController
type App struct {
	GorpController
}

#测试查询
func (c App) Index() revel.Result {
	greeting := "It works!"

	users, err := c.Txn.Select(models.User{}, `select * from user where Name="Hobo"`)
	if len(users) == 0 {
		greeting = "Null"
	} else {
		user := users[0].(*models.User)
		greeting = user.Name
	}
	if err != nil {
		panic(err)
	}
	return c.Render(greeting)
}

```

修改views/App/Index.html
```
#<h1>It works</h1>
<h1>/{/{.greeting/}/}</h1>		#去掉反斜杠
```

运行测试下
```
go run app_name
```



