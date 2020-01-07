---
title: "【Istio源码】Citadel & Node Agent"
date: 2020-01-07
lastmod: 2020-01-07
draft: false
mermaid: true
categories: [
	"Service Mesh",
]
tags: [
	"Service Mesh",
    "Istio",
    "源码"
]
---

Istio安全主要包括**认证**和**授权**，有关授权的`RBAC`使用参考之前的文章[【Istio安全】服务间访问控制-RBAC](/post/servicemesh/2019-03-09-istio-rbac-quick-start/#1-开启授权-clusterrbacconfig)
，不过`RBAC`已经[准备废弃](https://istio.io/docs/reference/config/security/istio.rbac.v1alpha1/)被`Authorization`替代，
**认证**则分为**服务间身份验证**和**来源身份认证**，服务间身份验证Istio提供了**双向TLS**方案，其中涉及的秘钥管理服务，主要由`istio/security`模块的
`Citadel`和`Node Agent`提供，本文主要分析这部分的源码实现。

<!--more-->

# PKI
首先参考[官方文档](https://preliminary.istio.io/zh/docs/concepts/security/#pki)[[en](https://istio.io/docs/concepts/security/#pki)]看下几种不同场景的方案，其中`Kubernetes 方案`最简单，
`Kubernetes 中的代理节点`更适合生产，具体优势[参考](https://preliminary.istio.io/zh/docs/tasks/security/citadel-config/auth-sds)[[en](https://istio.io/docs/tasks/security/citadel-config/auth-sds/)]。

> 
- Kubernetes 方案
	- Citadel 监视 Kubernetes apiserver，为每个现有和新的服务帐户创建 SPIFFE 证书和密钥对。Citadel 将证书和密钥对存储为 Kubernetes secret。
	- 创建 pod 时，Kubernetes 会根据其服务帐户通过 Kubernetes secret volume 将证书和密钥对挂载到 pod 上。
	- Citadel 监视每个证书的生命周期，并通过重写 Kubernetes secret 自动轮换证书。
	- Pilot 生成安全命名信息，该信息定义了哪些 Service Account 可以运行哪些服务。Pilot 然后将安全命名信息传递给 envoy sidecar。
- 本地机器方案
	- Citadel 创建 gRPC 服务来接受证书签名请求（CSR）。
	- 节点代理生成私钥和 CSR，并将 CSR 及其凭据发送给 Citadel 进行签名。
	- Citadel 验证 CSR 承载的凭证，并签署 CSR 以生成证书。
	- 节点代理将从 Citadel 接收的证书和私钥发送给 Envoy。
	- 上述 CSR 过程会定期重复进行证书和密钥轮换。
- Kubernetes 中的代理节点
	- Citadel 创建一个 gRPC 服务来接受 CSR 请求。
	- Envoy 通过 Envoy secret 发现服务（SDS）API 发送证书和密钥请求。
	- 收到 SDS 请求后，节点代理会创建私钥和 CSR，并将 CSR 及其凭据发送给 Citadel 进行签名。
	- Citadel 验证 CSR 中携带的凭证，并签署 CSR 以生成证书。
	- 节点代理通过 Envoy SDS API 将从 Citadel 接收的证书和私钥发送给 Envoy。
	- 上述 CSR 过程会定期重复进行证书和密钥轮换。

# CA管理-Citadel

Citadel(源码入口`istio/security/cmd/istio_ca`)是Istio自带的秘钥管理服务，使用代理节点也支持外部CA系统，如：VaultCA和GoogleCA。

包括以下能力：

- 证书管理:`pki/ca`
	- [IstioCA](#istioca)
- 工作负载的证书轮换:`k8s/controller`
	- [SecretController](#secretcontroller)
- 证书签名服务:`server/ca`
	- [Server](#server)
	
*TODO 配图*

## IstioCA
IstioCa支持两种方式：

- 自签名证书，自动轮换[SelfSignedCA](#selfsignedca)
- 外部文件证书
	
### SelfSignedCA
自签名证书，自动轮换管理由`SelfSignedCARootCertRotator`实现，默认`1小时`做一次证书的检查。

## SecretController
在Kubernetes方案中工作负载的证书管理，Citadel通过修改`secret`对工作负载的证书进行管理，`SecretController`监控`ServiceAccount`的创建、删除，
`Secret`的删除、更新，以及`Namespace`的更新，在产生变更时修改`secret`完成证书的轮换，同时`Pilot Agent`会监控`secret`的变更，并重启Envoy使配置生效，完成证书的轮换。

## Server
证书服务，与`nodeagent`的`caclient`交互，包括服务的认证及鉴权等

**gRPC服务如下:**

- CertificateService
	- CreateCertificate(context.Context, *IstioCertificateRequest) (*IstioCertificateResponse, error)
- CAService`废弃`
	- HandleCSR(context.Context, *CsrRequest) (*CsrResponse, error)

### Authenticator
gRPC服务认证，默认一个`ClientCertAuthenticator`，当启动SDS时添加了`KubeJWTAuthenticator`

### Authorizer
gRPC服务`CAService`和`CertificateService`的鉴权

> 服务的`Authorizer`都是`TODO`状态

**registry**提供了一个身份授权的注册表

- kube/ServiceController
	- 监控`Service`创建、删除与修改
- kube/ServiceAccountController
	- 监控`ServiceAccount`创建、删除与修改
	
# Node Agent K8S

NodeAgent(源码入口`istio/security/cmd/node_agent_k8s`)是Citadel在工作负载`node`或者网关`pod`中的代理，为Envoy提供`SDS`服务，并与Citadel交互发起`CSR`请求。
使用**Kubernetes中的代理节点**模式时Citadel可以运行为`server-only`模式，即不做工作负载的证书管理。

*TODO 配图*

> Node Agent源码均在`nodeagent`目录

## CA Client

提供CSR签名接口，服务实现支持`citadel`、`google`、`vault`

**Client接口**
```go
// Client interface defines the clients need to implement to talk to CA for CSR.
type Client interface {
	CSRSign(ctx context.Context, csrPEM []byte, subjectID string,
		certValidTTLInSec int64) ([]string /*PEM-encoded certificate chain*/, error)
}
```
	
## Secret Fetcher

作为Workload代理时根据不同的Provider实例化CA Client，或者作为Ingress Gateway代理时监控`secret`变更，并通知到`secretcache`

- Workload代理以`DaemonSet`在`node`上运行，使用CA Client与CA Server交互
- Ingress Gateway代理与Workload部署方式不同，代理在Pod内运行，而不是`DaemonSet`
	- 监控`secret`的创建、删除和更新，用于捕获Ingress Gateway的TLS配置
	- `secret`有变更时通过`AddCache`、`DeleteCache`和`UpdateCache`方法通知`secretcache`到`DeleteK8sSecret`和`UpdateK8sSecret`，再由`sds.NotifyProxy`通知SDS服务
	- [使用SDS的优势](https://preliminary.istio.io/zh/docs/tasks/traffic-management/ingress/secure-ingress-sds/#configure-a-TLS-ingress-gateway-using-sds)[[en](https://istio.io/docs/tasks/traffic-management/ingress/secure-ingress-sds/#configure-a-tls-ingress-gateway-using-sds)]
	- Ingress Gateway的TLS的默认配置方式是[文件挂载](https://preliminary.istio.io/zh/docs/tasks/traffic-management/ingress/secure-ingress-mount/)[[en](https://istio.io/docs/tasks/traffic-management/ingress/secure-ingress-mount/)]

## Secret Cache

秘钥缓存，负责秘钥的轮换

- 默认10min做一次检查，对即将过期的做轮换，默认1小时后过期的证书就做更新
	- 如果轮询时`token`过期会返回`secret=nil`，这时SDS服务在收到`notify`后需要将连接断开重连
- 产生轮换时通过`sds.NotifyProxy`通知到SDS服务，SDS服务通过`connKey`找到对应的`client`，并通过SDS的`StreamSecrets`将新的证书推送到`Pod`
- 在`generateSecret`时，如果检测到`rootCert`为`nil`或者与`certChainPEM`的不一致，将触发`rootCert`的更新
	- SDS服务将新的`rootCert`加入到可信`CA`

**SecretManager接口**
```go
// SecretManager defines secrets management interface which is used by SDS.
type SecretManager interface {
	// GenerateSecret generates new secret and cache the secret.
	GenerateSecret(ctx context.Context, connectionID, resourceName, token string) (*model.SecretItem, error)

	// ShouldWaitForIngressGatewaySecret indicates whether a valid ingress gateway secret is expected.
	ShouldWaitForIngressGatewaySecret(connectionID, resourceName, token string) bool

	// SecretExist checks if secret already existed.
	// This API is used for sds server to check if coming request is ack request.
	SecretExist(connectionID, resourceName, token, version string) bool

	// DeleteSecret deletes a secret by its key from cache.
	DeleteSecret(connectionID, resourceName string)
}
```

## SDS Service

SDS服务实现

- `NotifyProxy`接收`secret`变更的通知
	- 查找`client`的`connection`，通过`pushChannel`将`secret`变更事件分发给SDS服务接口`StreamSecrets`，然后推送到Envoy
	- 事件类型：
        - `Secret_TlsCertificate`TLS证书
        - `Secret_ValidationContext`可信CA
- 接收Envoy对SDS服务`StreamSecrets`和`FetchSecrets`请求，`SecretManager.GenerateSecret()`生成秘钥

# Ref

- [深度解析Istio系列之安全模块篇](https://zhuanlan.zhihu.com/p/59825498)
- [Istio 安全设置笔记](https://blog.fleeto.us/post/istio-security-notes/)
