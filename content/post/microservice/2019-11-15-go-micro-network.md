---
title: "【go-micro】Network初探"
date: 2019-11-15
lastmod: 2019-11-16 07:03:09
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
Network是micro社区正在主力打造的解决多"云"环境的解决方案，下面结合最近的研究做个总结，主要包括`network`适用的几种场景分析，
以及在这些场景需求下`network`还有哪些不足。

<!--more-->

- [Network Overview](https://micro.mu/docs/network.html)
- [Service Network](https://micro.mu/docs/service-network.html)


## Proxy
在分析`network`之前先了解`micro`的`proxy`模式，服务之间调用不需要服务发现，而是通过`proxy`做转发，由`proxy`完成服务的发现和调用。
`network`实现了一个特殊的代理，它的服务发现并不只是通过注册中心，同时有`network`之间共享的服务。
![micro-proxy](/img/micro/proxy.png)

## 服务可见性
`network`通过共享服务信息，实现跨`network`的服务可见，从而实现代理的服务发现，并且共享是有可见性的，当前`advertise_strategy`有`3+1`种方案:

- `none`不共享服务
	- 即只接受外部服务，适合边缘节点，依赖中心服务场景
- `all`共享全部服务
- `best`共享最优路由
	- `local`仅共享集群内部服务，而不共享从其他`network`共享得到的服务，实际是`best`的一个筛选项，这也是为什么说它是`3+1`
	
> 截止`v1.6.0`版本还没有此功能，需要使用`master`分支，[PR#932](https://github.com/micro/go-micro/pull/932)增加了此策略的支持，
以micro的发版速度应该要不了多久就会有Release版🤣

> Network在不断完善，本文参考版本为：
>
 - [micro/tree/98d9b1f332a2](https://github.com/micro/micro/tree/98d9b1f332a2)
 - [go-micro/tree/9f481542f38d](https://github.com/micro/go-micro/tree/9f481542f38d)
	
## 应用场景
### 多集群
多集群场景分两种一是**集群服务共享**，二是**多中心/主备集群**，在`advertise_strategy`策略选择有区别。

#### 集群服务共享
集群服务共享是指不同集群拥有不同的服务，实现集群间的服务发现，选择的策略是`advertise_strategy=local`，`network`仅共享自身服务，
如果`DC B`与`DC C`需要共享服务，那么需要直接建立连接，而不是通过`DC A`
![network_multi_cluster](/img/micro/network_multi_cluster_1.png)

#### 多中心/主备集群
多中心/主备集群是将同一套服务部署在多个中心，可能是多活，也可能是分主备，选择的策略是`advertise_strategy=best`，但现在`network`在负载策略上还没有**就近原则**
![network_multi_cluster](/img/micro/network_multi_cluster_2.png)

### 中心服务+边缘
对于边缘计算场景，边缘节点可能是不提供服务的，比如只上报数据，可以选择`advertise_strategy=none`，当然有的可能又是提供服务的，可以选择`advertise_strategy=local`，
但`DC A`并不会将`DC C`的服务共享给`DC B`。因为`network`间通过`tunnel`通信，边缘节点可以是任意网络，只要连接到中心服务的`network`即可。
![network_edge](/img/micro/network_edge.png)

## 不足
### Network高可用及性能
要考虑`network`的高可用，必须支持多富本水平扩展，现在同一集群内`network`会共享服务，但当前`network`的服务可见性并没有将集群内`node`与直连的`node`节点做区分，
如果集群内出现n个副本，那么最差的情况下可能要在`network`的`pod`间跳转n次。理想状态应该是对于集群内仅共享与`node`直连的**集群外服务**（这是当前`network`不具备），
而对集群外共享本集群服务，这样可以确保集群内`network`代理最多经过**两跳**即可调用集群外服务。
![network_edge](/img/micro/network_problem_ha.png)
图中红色标记的`go.micro.srv.s-2 gateway-a-2`、`go.micro.srv.s-2 gateway-a-3`不应该出现在`network`的`routes`中，避免同一集群内没必要的路由。

总的来说是`advertise_strategy`需要更完善的策略支持，做必要的隔离、筛选，一是服务的可见性问题，二是服务规模增加后`network`间的数据通信也会成为瓶颈。

### 路由负载策略
路由负载选择需要优先级、筛选等策略，如local优先，这样我们可以实现主备集群，以及集群间流量切换等场景的应用

### 安全
`network`间的连接是通过`tunnel`完成，而`tunnel`在安全方面只是在`message`的`header`中放了一个`token`用于相互验证，这也是需要加强的部分。

## Network模拟测试
可以本地使用不同的注册中心模拟多个集群，简单的`mdns`、`consul`和`etcd`就可以模拟出三个集群，如果有公网服务器当然更好。
