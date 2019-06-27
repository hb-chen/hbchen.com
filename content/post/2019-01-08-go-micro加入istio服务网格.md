---
title: "go-micro加入Istio服务网格"
date: 2019-02-21T16:59:37+08:00
comments: true
categories: [
	"微服务",
	"Service Mesh",
]
tags: [
	"微服务",
	"Service Mesh",
    "istio",
    "go-micro"
]
---

将go-micro服务加入service mesh，Client、Server不需要Registry、Selector、Transport等，通过自定义micro的server & client插件，去掉在istio中不需要的组件依赖。

<!--more-->

![go-micro](https://raw.githubusercontent.com/hb-go/micro/master/doc/img/micro-istio.png)

[hb-go/micro-plugins](https://github.com/hb-go/micro-plugins)实现了gRPC、http的Istio版本Plugin，下面介绍如何使用。

> 完整示例参考[hb-go/micro/istio](https://github.com/hb-go/micro/tree/master/istio)，示例包括http、gRPC

##### 命令行参数
方便服务运行时指定端口，在命令行获取服务server、client端口配置，参数根据具体情况自行设计

> 在服务网格中我倾向统一上下游服务端口，避免不必要的配置以及因此引发的冲突问题

- Client端服务地址`CallOptions.Address`解析规则：
    - `:`开头，将`service.Name`中`.`替换为`-`，加`CallOptions.Address`，如`go.micro.api.sample` `:9080` => `go-micro-api-sample:9080`
    - 非`:`开头，固定地址

```go
var (
	serverAddr string
	callAddr   string
	cmdHelp    bool
)

func init() {
	flag.StringVar(&serverAddr, "server_address", "0.0.0.0:9080", "server address.")
	flag.StringVar(&callAddr, "client_call_address", ":9080", "client call options address.")
	flag.Parse()
}
```

##### 自定义server、client插件，创建服务

> 由于micro框架对命令行的解析问题，创建服务时需要增加`micro.Flags(...)`，兼容自定义参数，如:`client_call_address`

```go
import (
	httpClient "github.com/hb-go/micro-plugins/client/istio_http"
	httpServer "github.com/hb-go/micro-plugins/server/istio_http"
)

func main() {
	c := httpClient.NewClient(
		client.ContentType("application/json"),
		func(o *client.Options) {
			o.CallOptions.Address = callAddr
		},
	)
	s := httpServer.NewApiServer(
		server.Address(serverAddr),
	)

	// New Service
	service := micro.NewService(
		micro.Name("go.micro.api.sample"),
		micro.Version("latest"),
		micro.Registry(noop.NewRegistry()),
		micro.Client(c),
		micro.Server(s),

		// 兼容micro cmd parse
		micro.Flags(cli.StringFlag{
			Name:   "client_call_address",
			EnvVar: "MICRO_CLIENT_CALL_ADDRESS",
			Usage:  " Invalid!!!",
		}),
	)

	service.Options().Cmd.Init()
	
	// ……
}
```

##### 服务部署.yaml

> 其它Istio相关.yaml参考完整示例[hb-go/micro/istio/k8s](https://github.com/hb-go/micro/tree/master/istio/k8s)

```yaml
######################################################################################
# API service
######################################################################################
apiVersion: v1
kind: Service
metadata:
  name: go-micro-api-sample
  labels:
    app: go-micro-api-sample
spec:
  ports:
  - port: 9080
    name: http
  selector:
    app: go-micro-api-sample
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: go-micro-api-sample-v1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: go-micro-api-sample
        version: v1
    spec:
      containers:
      - name: go-micro-api-sample
        command: [
          "/sample",
          "-server_address=0.0.0.0:9080",
          "-client_call_address=:9080",
        ]
        image: hbchen/go-micro-istio-api-sample:v0.0.5
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9080
---
```

