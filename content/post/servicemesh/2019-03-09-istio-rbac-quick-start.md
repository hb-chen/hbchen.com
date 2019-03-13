---
title: "Istio服务间访问控制-RBAC"
date: 2019-03-09T09:21:22+08:00
comments: true
categories: [
	"Service Mesh",
]
tags: [
	"Service Mesh",
    "Istio",
    "安全",
    "RBAC"
]
---

Istio提供了非常易用的安全解决方案，包括服务间身份验证`mTLS`，服务间访问控制`RBAC`，以及终端用户身份验证`JWT`等，接下来将介绍如何使用服务间访问控制，同时涉及`双向TLS`。
<!--more-->

> 
- Istio版本 **1.1.0-rc.3**
- 在的[github.com/hb-go/micro-mesh](https://github.com/hb-go/micro-mesh)中有结合示例的[RBAC配置实践](https://github.com/hb-go/micro-mesh/tree/master/deploy/k8s/rbac)可以参考


**要实现`RBAC`主要理解以下几个类型的`yaml`配置，以及之间的关系：**

- [双向TLS](#双向TLS)
    - `Policy`或`MeshPolicy`，上游`server`开启TLS
    - `DestinationRule`，下游`client`开启TLS
- [RBAC](#RBAC)
    - `ClusterRbacConfig`，启用授权及范围
    - `ServiceRole`，角色权限规则
    - `ServiceRoleBinding`，角色绑定规则
- [Optional](#Optional)
    - `ServiceAccount`，`RoleBinding``subjects`的`user`条件
    
![auth-adapter](https://raw.githubusercontent.com/hb-chen/hbchen.com/master/static/img/istio-tls-rbac.png)
    
## 双向TLS

> 
- [Istio文档-认证策略](https://preliminary.istio.io/zh/docs/concepts/security/#%E8%AE%A4%E8%AF%81)
    - [认证策略](https://preliminary.istio.io/zh/docs/concepts/security/#%E8%AE%A4%E8%AF%81%E7%AD%96%E7%95%A5)
- [Istio文档-基础认证策略](https://preliminary.istio.io/zh/docs/tasks/security/authn-policy/)
    - [为网格中的所有服务启用双向 TLS 认证](https://preliminary.istio.io/zh/docs/tasks/security/authn-policy/#%E4%B8%BA%E7%BD%91%E6%A0%BC%E4%B8%AD%E7%9A%84%E6%89%80%E6%9C%89%E6%9C%8D%E5%8A%A1%E5%90%AF%E7%94%A8%E5%8F%8C%E5%90%91-tls-%E8%AE%A4%E8%AF%81)

### 1.上游`server`开启TLS

***[策略范围说明](##)***

> 
- **网格范围策略**：在网格范围存储中定义的策略，没有目标选择器部分。网格中最多只能有一个网格范围的策略。
- **命名空间范围的策略**：在命名空间范围存储中定义的策略，名称为 default 且没有目标选择器部分。每个命名空间最多只能有一个命名空间范围的策略。
- **特定于服务的策略**：在命名空间范围存储中定义的策略，具有非空目标选择器部分。命名空间可以具有零个，一个或多个特定于服务的策略。

**`Policy`特定于服务的策略**

- `targets`支持`name`以及`ports`列表

```yaml
apiVersion: "authentication.istio.io/v1alpha1"
kind: "Policy"
metadata:
  name: "policy-name"
spec:
  targets:
  - name: service-name-1
  - name: service-name-2
    ports:
    - number: 8080
  peers:
  - mtls: {}
---
```

**`Policy`命名空间范围的策略**
```yaml
apiVersion: "authentication.istio.io/v1alpha1"
kind: "Policy"
metadata:
  name: "default"
  namespace: "namespace-1"
spec:
  peers:
  - mtls: {}
---
```

**`MeshPolicy`网格范围策略**
```yaml
apiVersion: "authentication.istio.io/v1alpha1"
kind: "MeshPolicy"
metadata:
  name: "default"
spec:
  peers:
  - mtls: {}
---
```

### 2.下游`client`开启TLS
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: service-name-1
spec:
  host: service-host-1
  # 开启TLS
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
  subsets:
  - name: v1
    labels:
      version: v1
---
```

## RBAC

>
- [Istio文档-授权](https://preliminary.istio.io/zh/docs/concepts/security/#%E6%8E%88%E6%9D%83)
    - [启用授权](https://preliminary.istio.io/zh/docs/concepts/security/#%E5%90%AF%E7%94%A8%E6%8E%88%E6%9D%83)
    - [授权策略](https://preliminary.istio.io/zh/docs/concepts/security/#%E6%8E%88%E6%9D%83%E7%AD%96%E7%95%A5)
- [Istio文档-基于角色的访问控制](https://preliminary.istio.io/zh/docs/tasks/security/role-based-access-control/)
    - [服务级的访问控制](https://preliminary.istio.io/zh/docs/tasks/security/role-based-access-control/#%E6%9C%8D%E5%8A%A1%E7%BA%A7%E7%9A%84%E8%AE%BF%E9%97%AE%E6%8E%A7%E5%88%B6)
- [Istio文档-迁移 RbacConfig 到 ClusterRbacConfig](https://preliminary.istio.io/zh/docs/setup/kubernetes/upgrade/#%E8%BF%81%E7%A7%BB-rbacconfig-%E5%88%B0-clusterrbacconfig)
    - *这里使用的`ClusterRbacConfig`*
    
有关`RbacConfig`、`ServiceRole`、`ServiceRoleBinding`的属性结构Istio文档有详细的配置可以参考[RBAC](https://preliminary.istio.io/zh/docs/reference/config/authorization/istio.rbac.v1alpha1/)

### 1.开启授权`ClusterRbacConfig`

```yaml
apiVersion: "rbac.istio.io/v1alpha1"
kind: ClusterRbacConfig
metadata:
  name: default
  namespace: istio-system
spec:
  mode: 'ON_WITH_INCLUSION'
  inclusion:
    namespaces: ["namespace-1"]
    services: ["service-name-1.namespace-1.svc.cluster.local", "service-name-2.namespace-1.svc.cluster.local"]
  # ENFORCED/PERMISSIVE，严格或宽容模式
  enforcement_mode: ENFORCED
---
```
### 2.角色权限规则`ServiceRole`
`namespace` + `services` + `paths` + `methods` 一起定义了如何访问服务，其中`services`必选，另外有`constraints`可以指定其它约束
```yaml
apiVersion: "rbac.istio.io/v1alpha1"
kind: ServiceRole
metadata:
  name: service-role-1
  namespace: default
spec:
  rules:
  - services: ["service-name-1.namespace-1.svc.cluster.local"]
    methods: ["*"]
    constraints:
    - key: request.headers[version]
      values: ["v1", "v2"]
---
```
### 3.角色绑定规则`ServiceRoleBinding`
`user` + `properties` 一起定义授权给谁
```yaml
apiVersion: "rbac.istio.io/v1alpha1"
kind: ServiceRoleBinding
metadata:
  name: service-rb-1
  namespace: default
spec:
  subjects:
  # NOTE:需要添加 ServiceAccount
  - user: "cluster.local/ns/namespace-1/sa/service-account-2"
  # ingressgateway
  - user: "cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"
    properties:
      source.namespace: "abc"
  roleRef:
    kind: ServiceRole
    name: "service-role-1"
---
```

## Optional

### 部署实例添加`ServiceAccount`
对于需要要在`ServiceRoleBinding`的`subjects`条件中授权的`user`，需要在部署实例时指定`serviceAccountName`

```yaml
# 创建ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: service-account-2
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: service-name-2-v1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: service-name-2
        version: v1
    spec:
      # 为部署实例指定serviceAccountName
      serviceAccountName: service-account-2
      containers:
      - name: service-name-2-v1
        command: [
          "/main"
        ]
        image: hbchen/service-2:v0.0.1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9080
---
```

