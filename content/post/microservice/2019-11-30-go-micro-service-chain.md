---
title: "【go-micro】微服务协作开发、灰度发布之流量染色"
date: 2019-11-30T16:40:32+08:00
lastmod: 2019-12-01
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

协作开发与灰度发布是微服务框架在流量治理能力方面的两个体现，本文结合**go-micro**实践对流量进行染色，实现开发环境的多分支协作，
以及生产环境的灰度发布。

<!--more-->

# 场景

## 开发环境多服务、多分支协作

![micro-chain-dev](/img/micro/chain-dev.png)

- `QA`组测试`v1.2`和`v2.0`链路
	- `v2.0` + `v1.2`链路
- `v1.1`组仅关注`v1.1`的版本开发
	- `v1.1` + `master`链路
- `v1.2`组在`v1.1`开发新版`srv-2`服务
	- `v1.2` + `v1.1` + `master`链路
- `v2.0`组仅关注`v2.0`的版本开发
	- `v2.0` + `master`链路
	
## 生产环境灰度发布

![micro-chain-gray](/img/micro/chain-gray.png)

- 普通用户
	- `Pro`链路
- 灰度测试用户
	- `Gray` + `Pro`链路
- QA
	- `Pre` + `Pro`链路

# 流量染色

**流量染色核心：**

- gateway对请求进行染色
	- 染色规则可以是`host`、`header`字段、`agent`终端信息、用户筛选、流量比例等等
- 染色信息在服务间传递
	- `go-micro`中`http`请求的`header`以及`rpc`请求的`metadata`
- 服务调用时根据染色信息对服务进行筛选，实现调用链路的管控

我们基于`go-micro`实践的是实现**多链路染色**，染色链路带有优先级，如[开发环境多服务、多分支协作](#开发环境多服务-多分支协作)的`v1.2`组，
虽然`v1.1`和`v1.2`都有`srv-2`服务，但我们在染色信息中`v1.2`在前优先选择，所以可以实现多分支同时染色(PS：如果两个分支中两个服务的优先级相反无法实现，需要设计更复杂的染色方案)

> 网关染色及`client wrapper`实现参考我实现的两个`chain`插件
>
 - [hb-chen/micro-plugin/micro/chain](https://github.com/hb-go/micro-plugins/tree/master/micro/chain)
 - [hb-chen/micro-plugin/wrapper/select/chain](https://github.com/hb-go/micro-plugins/tree/master/wrapper/select/chain)

## 染色
在网关对流量进行**染色**，基于`mciro`的插件，可以方便的实现，具体染色规则需要根据自身需求实现。
```bash
// 链路染色
api.Register(chain.New(chain.WithChainsFunc(func(r *http.Request) []string {
	return []string{"v2.0", "v1.2"}
})))

web.Register(chain.New(chain.WithChainsFunc(func(r *http.Request) []string {
	return []string{"v2.0"}
})))
```
## 调用链路管控
`go-micro`实现调用链路管控，最大的障碍就是**网关**，`API`及`Web`均不支持服务筛选，需要自己二次开发，相关问题也反馈给社区看后续计划[#1003](https://github.com/micro/go-micro/issues/1003)。

### 网关服务筛选-坑

**API网关**仅尝试了`api handler`，修改相对简单，只需要增加`ClientWrapper`，并去掉指定的负载策略即可。如果使用其他`handler`需要逐个解决。

*1. `micro/main.go`添加`ClientWrapper`*
```go
func main() {
	cmd.Init(
		// 链路染色
		micro.WrapClient(chain.NewClientWrapper()),
	)
}
```

*2. `go-micro/api/handler/api.go`去掉`strategy`，`rpc handler`类似*
```go
// create strategy
// so := selector.WithStrategy(strategy(service.Services))

// if err := c.Call(cx, req, rsp, client.WithSelectOption(so)); err != nil {
if err := c.Call(cx, req, rsp); err != nil {
	
}
```

**Web网关**相对麻烦，不能方便的使用`micro`的`options`，暂时测试在`web`模块实现`filter`

*1. `micro/web/web.go`修改`func (s *srv) proxy() http.Handler`，增加`SelectOption`*
```go
func (s *srv) proxy() http.Handler {
	sel := s.service.Client().Options().Selector

	director := func(r *http.Request) {
		kill := func() {
			r.URL.Host = ""
			r.URL.Path = ""
			r.URL.Scheme = ""
			r.Host = ""
			r.RequestURI = ""
		}

		parts := strings.Split(r.URL.Path, "/")
		if len(parts) < 2 {
			kill()
			return
		}
		if !re.MatchString(parts[1]) {
			kill()
			return
		}

		// NOTE: 添加SelectorOption
		val := r.Header.Get("X-Micro-Chain")
		chains := strings.Split(val, ";")

		next, err := sel.Select(Namespace+"."+parts[1], selector.WithFilter(filterChain(chains)))
		if err != nil {
			kill()
			return
		}

		s, err := next()
		if err != nil {
			kill()
			return
		}

		r.Header.Set(BasePathHeader, "/"+parts[1])
		r.URL.Host = s.Address
		r.URL.Path = "/" + strings.Join(parts[2:], "/")
		r.URL.Scheme = "http"
		r.Host = r.URL.Host
	}

	return &proxy{
		Default:  &httputil.ReverseProxy{Director: director},
		Director: director,
	}
}

// NOTE: SelectOption的filter实现
func filterChain(chains []string) selector.Filter {
	return func(old []*registry.Service) []*registry.Service {
		if len(chains) == 0 {
			return old
		}

		var services []*registry.Service

		chain := ""
		idx := 0
		for _, service := range old {
			serv := new(registry.Service)
			var nodes []*registry.Node

			for _, node := range service.Nodes {
				if node.Metadata == nil {
					continue
				}

				val := node.Metadata["chain"]
				if len(val) == 0 {
					continue
				}

				if len(chain) > 0 && idx == 0 {
					if chain == val {
						nodes = append(nodes, node)
					}
					continue
				}

				// chains按顺序优先匹配
				ok, i := inArray(val, chains)
				if ok && idx > i {
					// 出现优先链路，services清空，nodes清空
					idx = i
					services = services[:0]
					nodes = nodes[:0]
				}

				if ok {
					chain = val
					nodes = append(nodes, node)
				}
			}

			// only add service if there's some nodes
			if len(nodes) > 0 {
				// copy
				*serv = *service
				serv.Nodes = nodes
				services = append(services, serv)
			}
		}

		if len(services) == 0 {
			return old
		}

		return services
	}
}

func inArray(s string, d []string) (bool, int) {
	for k, v := range d {
		if s == v {
			return true, k
		}
	}
	return false, 0
}
```
### 服务筛选

普通服务低啊用的筛选框架支持没有什么问题，首先要为服务添加`metadata`信息，不赘述。其次要对`Client`进行包装，服务调用时添加`SelectOption`，
实现参考我的[chain](https://github.com/hb-go/micro-plugins/tree/master/wrapper/select/chain)插件。


*1. 添加`metadata`*
```go
md := make(map[string]string)
md["chain"] = "v2.0"

// New Service
service := micro.NewService(
	micro.Name("go.micro.api.xxx"),
	micro.Version("latest"),
	micro.Metadata(md),
)
```

*2. Wrap Client*

> 需要注意的是`service`初始过程，有类似`micro new`模块将service client注入到context的方法，需要分两次Init()，先WrapClient，再WrapHandler，
否则注入的`client`将是未被包装的。这也是micro比较常遇到的**坑**！

```go
// Initialise service
// NOTE: 注意有类似micro new模块将service client注入到context的方法，需要分两次Init()，否则注入的Client并未被包装
service.Init(
	micro.WrapClient(chain.NewClientWrapper()),
)
service.Init(
	// create wrap for the Example srv client
	micro.WrapHandler(client.AccountWrapper(service)),
)
```


# Ref

- [大规模微服务场景下灰度发布与流量染色实践](https://mp.weixin.qq.com/s/UBoRKt3l91ffPagtjExmYw)
