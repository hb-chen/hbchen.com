---
title: "【go-micro】开发协作-本地服务接入线上集成环境"
date: 2020-03-22T10:53:43+08:00
lastmod: 2020-03-22T10:53:43+08:00
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

在[微服务协作开发、灰度发布之流量染色](/post/microservice/2019-11-30-go-micro-service-chain/)介绍了如何通过流量染色在开发环境进行多服务、多版本的协作开发，
但这都是在服务已经发布到集成环境情况下，开发过程中对于单个服务的开发/维护者往往需要快速的将本地服务集成到在线环境，以完成更好集成测试，而不是等待发布流程，
这样一是节约时间，二是避免将有问题的服务发布到线上集成环境，对Ta人造成影响。本文将介绍在`micro`生态如何通过`network`将本地服务加入到线上集成环境，既可以快速测试，又不影响集成环境。

<!--more-->

# 方案

![network_dev_local](/img/micro/dev-local.png)

整体方案一是`go-micro`的服务的`proxy`方式，，并且在代理中有`router`筛选功能，二是`network`提供连接不同网络环境的能力。

1. 首先在线上和本地部署`network`服务，有关`network`的内容可以参考本文[【go-micro】Network初探](/post/microservice/2019-11-15-go-micro-network/)，
有了`network`的基础剩下的工作就是`proxy`的`router`筛选功能，实现参考 PR[#897](https://github.com/micro/go-micro/pull/897)，
从实现我们知道通过`Micro-Router`、`Micro-Network`和`Micro-Gateway`三个`metadata`可以对`router`进行筛选，而使用`Micro-Address`以及基于`label`的筛选还处于***TODO***状态。 

2. 其次在线上及本地所有服务都需要是用`network`做为代理(`MICRO_PROXY=go.micro.network`)包括网关(*注意默认网关有不支持服务筛选问题*)，这样服务间所有流量都通过`network`进行代理转发。

3. 最后考虑所有服务都可以自助路由到个人本地，我们不能直接使用`Micro-Router`，因为`Micro-Router`会在全链路生效，需要使用`metadata`来传递`router`筛选信息，
再通过`Client/Call Wrap`实现`Micro-Router`的恢复和清理，在识别到调用服务需要筛选时添加`Micro-Router`，否则清理`Micro-Router`，避免向下传导，实践参考[router_filter](https://github.com/micro-in-cn/starter-kit/tree/master/pkg/plugin/wrapper/client/router_filter)

***实践参考*** [micro-in-cn/starter-kit#本地服务接入-network代理](https://github.com/micro-in-cn/starter-kit#%E6%9C%AC%E5%9C%B0%E6%9C%8D%E5%8A%A1%E6%8E%A5%E5%85%A5-network%E4%BB%A3%E7%90%86)

# 局限性/问题

- 所有服务都要使用`network`做代理，如果框架能够在`CallOption`中支持动态使用代理将会更好
- 所有服务都需要添加`Client/Call Wrap`，需要统一的规范约定
- 在网关层与[流量染色](/post/microservice/2019-11-30-go-micro-service-chain/#%E7%BD%91%E5%85%B3%E6%9C%8D%E5%8A%A1%E7%AD%9B%E9%80%89-%E5%9D%91)遇到相同的问题即：***不支持服务筛选***，需要去掉`SelectOption`
- 为了避免误协作过程中路由到Ta人本地环境，可以再增加`Micro-Network`规则，线上与本地`network`使用不同名称(*未实测*)
    - *如果框架能够在路由选择上有`route.Link=local`这样的本地优先策略会有一个更好的体验*
