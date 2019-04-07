---
title: "【Istio源码】Pilot Agent"
date: 2019-04-07
lastmod: 2019-04-07
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
接上一篇[《【Istio源码】Pilot Discovery》](/post/servicemesh/2019-03-17-istio-code-pilot-discovery/)，这篇继续分析**Pilot**的`pilot-agent`模块。

<!--more-->
`pilot-agent`相对`pilot-discovery`在控制平面比较简单，**agent**更多的能力是在数据平面的`envoy`，`envoy`通过`xDS`协议与`pilot-discovery`服务进行通信。agent主要负责监控`ingress`、`mTLS`的证书，发生变更后重启`envoy`，以及提供**agent**状态的HTTP服务(*可选服务*)

```mermaid
sequenceDiagram
    participant a as agent
    participant pa as proxy/agent
    participant ew as envoy/watcher
    participant ep as envoy/proxy
    participant s as status
    participant fw as fsnotify.Watcher 
    
    Note over a,fw: agent状态服务
    a ->> s: NewServer()
    s -->> a: statusServer *Server
    a ->> s: statusServer.Run()

    a ->> ep: NewProxy(proxyConfig)
    ep -->> a: envoyProxy proxy.Proxy
    a ->> pa: NewAgent(envoyProxy)
    pa --> a: agent Agent
    a ->> ew: NewWatcher(agent.ConfigCh)，updates = agent.ConfigCh
    ew -->> a: watcher Watcher
    
    Note over a,fw: watcher监控配置变更，通知agent重启envoy
    loop: go
        a ->> ew: watcher.Run()
        ew ->> fw: fw.Watch(certs)，AuthCertsPath、IngressCertsPath
        fw -->> ew: w.watchFileEvents()
        ew ->> ew: <-timeChan，10s延迟
        ew ->> ew: SendConfig()
        ew ->> pa: updates<-
        Note over pa: <-configCh
    end    
    
    Note over a,fw: agent做envoy重启操作，包括频率控制、失败重试等
    loop : go
        a ->> pa: agent.Run()
        Note over pa: rateLimiter.Wait()<br/>控制频率
        alt <-configCh
            Note over pa: 配置更新Envoy热重启<br/>reconcile()
        else <-statusCh  
            Note over pa: Envoy重启后通知处理<br/>失败重试逻辑
        else <-reconcileTimer.C
            Note over pa: 失败重试计时重启<br/>reconcile()
        end
    end
    
    loop: envoy重启
        pa ->> pa: reconcile()
        Note over ep: 默认服务发现地址:<br/>istio-pilot:15010<br/>对应pilot-discovery的地址:<br/>gRPC port:15010
        pa ->> ep: proxy.Run(<-abort)
        ep -->> pa: err error
        pa ->> pa: statusCh<-
    end
    
```
