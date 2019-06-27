---
title: "【go-micro】使用kubernetes注册中心"
date: 2019-06-27
lastmod: 2019-06-27
draft: false
mermaid: false
comments: true
categories: [
	"微服务",
]
tags: [
	"go-micro",
    "微服务",
]
---

`go-micro`部署到`kubernetes`环境，可以选择`kubernetes`注册中心插件，减少组件依赖简化运维。
<!--more-->
# 主要工作
> 部署示例[hb-go/micro](https://github.com/hb-go/micro/tree/master/k8s)

- `micro`选择自己编译，而不是直接`go get -u github.com/micro/micro`
    - 自定义选择插件的支持
    - 自己发布镜像，官方`microhq/micro`上的镜像没有版本，容易出现兼容问题
    - 另外非常重要的一点**保证本地与线上`micro`的一致**，只需要替换注册中心`--registry=kubernetes`

- `go build`打包服务时增加`kubernetes`插件
```go
import (
	_ "github.com/micro/go-plugins/registry/kubernetes"
)
```

# RBAC问题
如果`kubernetes`开启了`RBAC`，在部署服务时需要配置`RBAC`，包括`micro web`、`micro api`服务，否则服务注册/发现将失败
```bash
2019/06/27 12:54:13 K8s: request failed with code 403
2019/06/27 12:54:13 K8s: request failed with body:
2019/06/27 12:54:13 {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"pods \"micro-web-79545546b4-p5vbt\" is forbidden: User \"system:serviceaccount:default:default\" cannot patch resource \"pods\" in API group \"\" in the namespace \"default\"","reason":"Forbidden","details":{"name":"micro-web-79545546b4-p5vbt","kind":"pods"},"code":403}
2019/06/27 12:54:13 Server register error: K8s: error
```

## RBAC yaml
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: micro-services
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: micro-registry
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - list
  - patch
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: micro-registry
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: micro-registry
subjects:
- kind: ServiceAccount
  name: micro-services
  namespace: default
```

## 服务指定Service Account
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  namespace: default
  name: micro-api
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: micro-api
    spec:
+     serviceAccountName: micro-services
      containers:
      - name: api
        command: [
          "/micro",
          "--registry=kubernetes",
          "--server=rpc",
          "--broker=http",
          "--transport=http",
          "--register_ttl=60",
          "--register_interval=30",
          "--selector=cache",
          "--enable_stats",
          "api"
        ]
        image: hbchen/micro:k8s
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
          name: api-port
```