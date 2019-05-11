---
title: "【Istio安全】网格边缘-Ingress"
date: 2019-05-11
draft: false
categories: [
	"Service Mesh",
]
tags: [
	"Service Mesh",
    "Istio",
    "安全",
]
---

接[【Istio安全】网格边缘-Egress]((/post/servicemesh/2019-04-11-istio-security-egress/))继续网格边缘的实践`ingress-gateway`，
分为**HTTP**、**HTTPS-不终止TLS**和**HTTPS-种子TLS**三种场景。

## 准备工作
有了[Egress](/post/servicemesh/2019-04-11-istio-security-egress/)的实践，这里不使用`httpbin`做内部服务，而是用`egress-gateway`的成果，
`www.aliyun.com`和`hbchen.com`的两个外部服务进行测试。

```yaml
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: entry-x
spec:
  hosts:
  - www.aliyun.com
  - hbchen.com
  ports:
  - number: 80
    name: http
    protocol: HTTP
  resolution: NONE
EOF

$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: originate-tls-for-aliyun
spec:
  host: www.aliyun.com
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
  subsets:
    - name: originate-tls
      trafficPolicy:
        loadBalancer:
          simple: ROUND_ROBIN
        portLevelSettings:
        - port:
            number: 443
          tls:
            mode: SIMPLE
EOF
```

**环境变量**
```bash
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')

export INGRESS_HOST=$(minikube ip)
```

> 接下来进入正文

都只需要定义两个`.yaml`配置:

- **Gateway**:`ingress-example-gateway`
- **VirtualService**:`ingress-example-svc`

## Http Gateway

![ServiceEntry](/img/istio/ingress-http.png)

> 官方文档[控制 Ingress 流量](https://istio.io/zh/docs/tasks/traffic-management/ingress/)

```yaml
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ingress-example-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: http
    hosts:
    - "hbchen.com"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ingress-example-svc
spec:
  hosts:
  - "hbchen.com"
  gateways:
  - ingress-example-gateway
  http:
  - match:
    - port: 80
    route:
    - destination:
        host: hbchen.com
        port:
          number: 80
EOF
```
**访问测试**
```bash
curl -I -v -HHost:hbchen.com http://$INGRESS_HOST:$INGRESS_PORT
```

## Https Gateway-不终止`TLS`

![ServiceEntry](/img/istio/ingress-https-passthrough.png)

> 官方文档[没有 TLS 的 Ingress gateway](https://istio.io/zh/docs/examples/advanced-gateways/ingress-sni-passthrough/)

不终止`TLS`即透传`https`请求到网格内服务，这里我们使用一个外部服务来模拟一个支持`https`的服务

```yaml
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ingress-example-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: http
    hosts:
    - "www.aliyun.com"
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: PASSTHROUGH
    hosts:
    - "www.aliyun.com"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ingress-example-svc
spec:
  hosts:
  - "www.aliyun.com"
  gateways:
  - ingress-example-gateway
  http:
  - match:
    - port: 80
    route:
    - destination:
        host: www.aliyun.com
        subset: originate-tls
        port:
          number: 443
  tls:
  - match:
    - port: 443
      sni_hosts:
      - www.aliyun.com
    route:
    - destination:
        host: www.aliyun.com
        port:
          number: 443
EOF
```

**访问测试**
```bash
curl -I -v -HHost:www.aliyun.com http://$INGRESS_HOST:$INGRESS_PORT
curl -I -v --resolve www.aliyun.com:$SECURE_INGRESS_PORT:$INGRESS_HOST https://www.aliyun.com:$SECURE_INGRESS_PORT
```

## Https Gateway-终止`TLS`

![ServiceEntry](/img/istio/ingress-https.png)

> 官方文档[使用 SDS 为 Gateway 提供 HTTPS 加密支持](https://istio.io/zh/docs/tasks/traffic-management/secure-ingress/sds/)

终止`TLS`即由`ingress-gateway`提供`https`服务，更符合`ingress-gateway`作为网格统一入口的需求

### 启用SDS
`SDS`默认是关闭的，所以首先需要开启`ingress-gateway`的`SDS`
```bash
$ cd istio/path 

# 生成.yaml
$ helm template install/kubernetes/helm/istio/ --name istio \
--namespace istio-system -x charts/gateways/templates/deployment.yaml \
--set gateways.istio-egressgateway.enabled=false \
--set gateways.istio-ingressgateway.sds.enabled=true > \
$HOME/istio-ingressgateway.yaml

# 重新部署
$ kubectl apply -f $HOME/istio-ingressgateway.yaml
```

### 生成证书
```bash
$ git clone https://github.com/nicholasjackson/mtls-go-example

$ cd mtls-go-example
$ ./generate.sh hbchen.com 123456

$ mkdir ./../hbchen.com && mv 1_root 2_intermediate 3_application 4_client ./../hbchen.com
$ cd ..
```

### 创建`Secret`
```bash
$ kubectl create -n istio-system secret generic hbchen-credential \
--from-file=key=hbchen.com/3_application/private/hbchen.com.key.pem \
--from-file=cert=hbchen.com/3_application/certs/hbchen.com.cert.pem
```

```yaml
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ingress-example-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: "hbchen-credential" # NOTE: 和 Secret 名称一致
    hosts:
    - "hbchen.com"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ingress-example-svc
spec:
  hosts:
  - "hbchen.com"
  gateways:
  - ingress-example-gateway
  http:
  - match:
    - port: 443
    route:
    - destination:
        host: hbchen.com
        port:
          number: 80
EOF
```

**访问测试**
```bash
curl -I -v -HHost:hbchen.com \
--resolve hbchen.com:$SECURE_INGRESS_PORT:$INGRESS_HOST \
--cacert hbchen.com/2_intermediate/certs/ca-chain.cert.pem \
https://hbchen.com:$SECURE_INGRESS_PORT
```

## TODO
### JWT
### Authn