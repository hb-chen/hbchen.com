---
title: "【Istio安全】网格边缘Ingress、Egress"
date: 2019-04-11T16:08:16+08:00
draft: false
---
接上一篇《[【Istio安全】服务间访问控制-RBAC](/post/servicemesh/2019-03-09-istio-rbac-quick-start/)》，本文主要介绍与网格边缘Ingress、Egress相关的安全问题，加密传输`TLS`以及身份认证`JWT`、`Authn`

<!--more-->

> 
- Istio版本 **1.1.1**


## Egress HTTPS

### 1.开始前的准备
Istio部署配置的`global.outboundTrafficPolicy`参数，在1.1开始默认`ALLOW_ANY`，这种情况下如果不配置`ServiceEntry`，访问外包HTTP流量返回`404`，而HTTPS流量可以正常访问，为了测试`ServiceEntry`配置效果，将配置改为`REGISTRY_ONLY`，方便观察`ServiceEntry`配置的效果

> 本文示例Istio使用helm的values-istio-demo.yaml配置安装

修改`values-istio-demo.yaml`
```yaml
  outboundTrafficPolicy:
    mode: REGISTRY_ONLY
```
重新部署Istio
```bash
$ helm template install/kubernetes/helm/istio --name istio --namespace istio-system --values install/kubernetes/helm/istio/values-istio-demo.yaml | kubectl apply -f -
```

部署测试用例`sleep`
```bash
$ kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.1/samples/sleep/sleep.yaml

$ export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
```

### 2.ServiceEntry方式
先定义一个`ServiceEntry`，开启`www.aliyun.com`、`hbchen.com`的外部访问
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
```

访问测试
```bash
$ kubectl exec -it $SOURCE_POD -c sleep -- curl -sL -o /dev/null -D - http://hbchen.com
HTTP/1.1 200 OK

$ kubectl exec -it $SOURCE_POD -c sleep -- curl -sL -o /dev/null -D - http://www.aliyun.com
HTTP/1.1 301 Moved Permanently
...
command terminated with exit code 35

$ kubectl exec -it $SOURCE_POD -c sleep -- curl -sL -o /dev/null -D - https://www.aliyun.com
command terminated with exit code 35
```

只开`80`端口，`HTTP`正常，而对于`HTTPS`以及有`HTTP`重定向`HTTPS`的请求仍会失败`command terminated with exit code 35`，所以实践中外部服务一般同时开启`80`、`443`端口

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
    name: http-port
    protocol: HTTP
  - number: 443
    name: https-port
    protocol: HTTPS
  resolution: NONE
EOF
```

访问测试
```bash
$ kubectl exec -it $SOURCE_POD -c sleep -- curl -sL -o /dev/null -D - http://www.aliyun.com
HTTP/1.1 301 Moved Permanently
...
HTTP/2 200
...

$ kubectl exec -it $SOURCE_POD -c sleep -- curl -sL -o /dev/null -D - https://www.aliyun.com
HTTP/2 200
...
```

这时我们看到另一个问题，需要升级`HTTPS`的`HTTP`请求每次都要`301`重定向一次，在不修改代码的情况下可以在`proxy`将`HTTP`请求升级为`HTTPS`，避免二次跳转，需要加`VirtualService` + `DestinationRule`来实现。

在官方示例[出口流量的-tls](https://istio.io/zh/docs/examples/advanced-gateways/egress-tls-origination/#%E5%87%BA%E5%8F%A3%E6%B5%81%E9%87%8F%E7%9A%84-tls)中，仅将`HTTP`升级到`HTTPS`，而对正常的`HTTPS`没有支持，这里稍作改动，在`VirtualService`中为`80`端口的`destination`定义`subset`，这样在`DestinationRule`中默认`trafficPolicy`不做处理，仅对指定的`subset`做`HTTPS`转发，使网格内无论发起`HTTP`还是`HTTPS`的外网请求都可以正常访问。

```yaml
kubectl apply -f - <<EOF
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
    name: http-port
    protocol: HTTP
  - number: 443
    name: https-port
    protocol: HTTPS
  resolution: DNS
  location: MESH_EXTERNAL
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: rewrite-port-for-entry
spec:
  hosts:
  - www.aliyun.com
  http:
  - match:
      - port: 80
    route:
    - destination:
        subset: originate-tls
        host: www.aliyun.com
        port:
          number: 443
  tls:
  - match:
      - port: 443
        sniHosts:
        - www.aliyun.com
    route:
    - destination:
        host: www.aliyun.com
        port:
          number: 443
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: originate-tls-for-entry
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

访问测试
```bash
$ kubectl exec -it $SOURCE_POD -c sleep -- curl -sL -o /dev/null -D - http://hbchen.com
HTTP/1.1 200 OK

$ kubectl exec -it $SOURCE_POD -c sleep -- curl -sL -o /dev/null -D - http://www.aliyun.com
HTTP/1.1 200 OK

$ kubectl exec -it $SOURCE_POD -c sleep -- curl -sL -o /dev/null -D - https://www.aliyun.com
HTTP/2 200
```


### 3.Gateway方式

> TODO<br/>官方示例[配置 Egress gateway](https://istio.io/zh/docs/examples/advanced-gateways/egress-gateway/)

### 4.直接调用
> Istio文档[直接调用外部服务](https://istio.io/zh/docs/tasks/traffic-management/egress/#%E7%9B%B4%E6%8E%A5%E8%B0%83%E7%94%A8%E5%A4%96%E9%83%A8%E6%9C%8D%E5%8A%A1)

需要修改helm的values配置更新部署。并且要让新的`istio-sidecar-injector`生效，重新部署`sleep`用例。这种方式在Sidecar的Proxy中跳过了外部IP，虽然也有`excludeIPRanges`方式，但修改麻烦需要更新sidecar，并且这样也失去了Istio的其它能力，所以不怎么实用。
```yaml
  proxy:
    # Minikube
    includeIPRanges: "10.0.0.1/24"
```

## Gateway HTTPS

> TODO

SDS服务

## Gateway JWT

> TODO

## Gateway Authn

> TODO