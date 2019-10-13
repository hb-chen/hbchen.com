---
title: "【go-micro】微服务快速开始"
date: 2019-10-13T15:27:12+08:00
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

[github.com/hb-go/micro-quick-start](https://github.com/hb-go/micro-quick-start)

这只是一个简易教程，是结合演示文稿[micro-quick-start.pdf](https://github.com/hb-go/micro-quick-start/blob/master/micro-quick-start.pdf)的代码示例，
结合演示文稿可以快速了解微服务与`go-micro`框架，并可以结合代码示例进行验证测试，文稿中有关于[`micro`自定义](#micro自定义)以及`go-micro`现成组件的使用这里并没有全部示范，
详细参考文稿，本教程每个示例均对应一个`commit`，可以快速查看/比较示例相关代码，包含以下几个阶段：

<!--more-->

- [服务创建](https://github.com/hb-go/micro-quick-start/commit/e76368bbee7dec646d13f1d9644b9d529e88074f)
	- 通过`micro new`创建服务后进行完善，使模板创建的服务可以运行
- [自定义Server](https://github.com/hb-go/micro-quick-start/commit/5f7de50b10c19e19ef546144d6a11adf69ad1cfd)
- [自定义Transport](https://github.com/hb-go/micro-quick-start/commit/ff70f21ba9d6a9bbf889598c0837e058706bf2fd)
- [服务筛选](https://github.com/hb-go/micro-quick-start/commit/615d4604e251bf1188fe12a0be22e05ca9c4ebf9)
- [自定义Wrapper-监控](https://github.com/hb-go/micro-quick-start/commit/703851f4364c541c436896f85badb836859c3b7e)
- [超时&重试](https://github.com/hb-go/micro-quick-start/commit/2f0baa1f5117d44809be07d6435fe36aeb1b9990)
- [限流&熔断](https://github.com/hb-go/micro-quick-start/commit/11453df1566721c1b8aa50d253e3454986aa2c4f)
- [配置-consul](https://github.com/hb-go/micro-quick-start/commit/8573ae85a7bd70038c0e0a18900bfa76d4cddb6c)
- [链路追踪](https://github.com/hb-go/micro-quick-start/commit/d3f79f6c427822c1646ee7ccb2a4ba68c81681fd)
- [k8s部署](https://github.com/hb-go/micro-quick-start/commit/8979073c7f473635e84d0300b61a749b66bc3bd1)

## micro自定义
`micro`做为网关往往需要自定义，`micro`提供了`plugin`的自定义，以下是部分参考，如增加组件`tcp`、`kubernetes`，使用`go-plugins`中已有的`micro/metrics`插件，
以及完全自定义一个`metrics`插件。
```go
package main

import (
	"github.com/micro/go-micro/util/log"
	"github.com/micro/go-plugins/micro/metrics"
	"github.com/micro/micro/api"
	"github.com/micro/micro/plugin"
	"github.com/micro/micro/web"
	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
	"net/http"
	"strconv"

	// tcp transport
	_ "github.com/micro/go-plugins/transport/tcp"

	// k8s registry
	_ "github.com/micro/go-plugins/registry/kubernetes"
)

// Metrics
func init() {
	api.Register(metrics.NewPlugin())
	web.Register(metrics.NewPlugin())
}

// 自定义Metrics
func init() {
	api.Register(plugin.NewPlugin(
		plugin.WithHandler(
			func(h http.Handler) http.Handler {
				return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
					r.Header.Get("")
				})
			}),
	))

	api.Register(plugin.NewPlugin(
		plugin.WithHandler(
			func(h http.Handler) http.Handler {
				md := make(map[string]string)

				opsCounter := prometheus.NewCounterVec(
					prometheus.CounterOpts{
						Namespace: "micro",
						Name:      "request_total",
						Help:      "How many go-micro requests processed, partitioned by method and status",
					},
					[]string{"path", "method", "code"},
				)

				timeCounterSummary := prometheus.NewSummaryVec(
					prometheus.SummaryOpts{
						Namespace: "micro",
						Name:      "upstream_latency_microseconds",
						Help:      "Service backend method request latencies in microseconds",
					},
					[]string{"path", "method"},
				)

				timeCounterHistogram := prometheus.NewHistogramVec(
					prometheus.HistogramOpts{
						Namespace: "micro",
						Name:      "request_duration_seconds",
						Help:      "Service method request time in seconds",
					},
					[]string{"path", "method"},
				)

				reg := prometheus.NewRegistry()
				wrapreg := prometheus.WrapRegistererWith(md, reg)
				wrapreg.MustRegister(
					opsCounter,
					timeCounterSummary,
					timeCounterHistogram,
				)

				prometheus.DefaultGatherer = reg
				prometheus.DefaultRegisterer = wrapreg

				return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
					// 拦截metrics path，默认"/metrics"
					if r.URL.Path == "/metrics" {
						promhttp.Handler().ServeHTTP(w, r)
						return
					}

					path := r.URL.Path
					method := r.Method
					timer := prometheus.NewTimer(prometheus.ObserverFunc(func(v float64) {
						us := v * 1000000 // make microseconds
						timeCounterSummary.WithLabelValues(path, method).Observe(us)
						timeCounterHistogram.WithLabelValues(path, method).Observe(v)
					}))
					defer timer.ObserveDuration()

					ww := wrapWriter{ResponseWriter: w}
					h.ServeHTTP(&ww, r)
					log.Logf("statusCode: %d, %s", ww.StatusCode, strconv.Itoa(ww.StatusCode))
					opsCounter.WithLabelValues(path, method, strconv.Itoa(ww.StatusCode)).Inc()
				})
			}),
	))

}

type wrapWriter struct {
	StatusCode int
	http.ResponseWriter
}

func (ww *wrapWriter) WriteHeader(statusCode int) {
	log.Logf("statusCode: %d", statusCode)
	ww.StatusCode = statusCode
	ww.ResponseWriter.WriteHeader(statusCode)
}
```