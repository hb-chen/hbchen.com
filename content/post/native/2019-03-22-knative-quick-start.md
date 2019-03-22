---
title: "【Knative】Quick Start"
date: 2019-03-22T20:15:14+08:00
lastmod: 2019-03-22
draft: true
comments: true
categories: [
	"Native",
]
tags: [
	"Knative",
]
---
Knative快速开始示例

- [x] hello-world
- [x] autoscale-go
- [ ] source-to-url-go 

<!--more-->

**部署环境**

> 
- GKE **1.12.5-gke.10**
- Istio **1.1.0**

## 安装
Knative的全量安装组件需要的资源太多，开始测试还是选择自定义安装比较好，结合目标示例选择了以下组件：

- Serving Component:
    - https://github.com/knative/serving/releases/download/v0.4.0/serving.yaml
    - https://github.com/knative/serving/releases/download/v0.4.0/monitoring-metrics-prometheus.yaml
- Build Component:
    - https://github.com/knative/build/releases/download/v0.4.0/build.yaml
- Cluster roles:
    - https://raw.githubusercontent.com/knative/serving/v0.4.0/third_party/config/build/clusterrole.yaml
    
|安装文件名|	说明 | 依赖关系
|:---|:---|:---|
| serving.yaml | Installs the Serving component. | Cluster roles enabled, if interacting with Build
| monitoring-metrics-prometheus.yaml | Installs only Prometheus* | Serving component
| build.yaml | Installs the Build component.| Cluster roles enabled, if interacting with Serving
| clusterrole.yaml | Enables the Build and Serving components to interact. | Serving component, Build component

**Knative安装**
```bash
kubectl apply --filename https://github.com/knative/serving/releases/download/v0.4.0/serving.yaml \
--filename https://github.com/knative/serving/releases/download/v0.4.0/monitoring-metrics-prometheus.yaml \
--filename https://github.com/knative/build/releases/download/v0.4.0/build.yaml \
--filename https://raw.githubusercontent.com/knative/serving/v0.4.0/third_party/config/build/clusterrole.yaml
```

**安装情况**
```bash
kubectl get pods --namespace knative-serving
NAME                          READY     STATUS    RESTARTS   AGE
activator-56555cfd8f-gj9cq    2/2       Running   2          1m
autoscaler-6d768b4f67-c5lld   2/2       Running   2          1m
controller-6cf6f85976-4rpf2   1/1       Running   0          1m
webhook-6f7dfd6946-7q87q      1/1       Running   0          1m

kubectl get pods --namespace knative-build
NAME                                READY     STATUS    RESTARTS   AGE
build-controller-755f6dd8b4-nbdr2   1/1       Running   0          32s
build-webhook-588dcc4f7f-7pcgv      1/1       Running   0          32s

kubectl get pods --namespace knative-monitoring
NAME                                  READY     STATUS    RESTARTS   AGE
grafana-7c5b4595b8-qc5fn              1/1       Running   0          1m
kube-state-metrics-65648b6bf9-q9f5h   4/4       Running   0          1m
node-exporter-7fhnm                   2/2       Running   0          1m
node-exporter-9vn9t                   2/2       Running   0          1m
node-exporter-sd8kt                   2/2       Running   0          1m
node-exporter-vzrw4                   2/2       Running   0          1m
prometheus-system-0                   1/1       Running   0          48s
prometheus-system-1                   1/1       Running   0          48s
```

## hello-world
**`service.yaml`配置**
```yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: helloworld-go
  namespace: default
spec:
  runLatest:
    configuration:
      revisionTemplate:
        spec:
          container:
            image: gcr.io/knative-samples/helloworld-go
            env:
            - name: TARGET
              value: "Go Sample v1"
```
**部署**
```bash
kubectl apply --filename service.yaml
```

**测试**
```bash
# ingressgateway ip
kubectl get svc istio-ingressgateway --namespace istio-system
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                                                                                                                                      AGE
istio-ingressgateway   LoadBalancer   10.39.249.196   35.192.111.18   80:31380/TCP,443:31390/TCP,31400:31400/TCP,15029:31828/TCP,15030:30781/TCP,15031:32757/TCP,15032:31758/TCP,15443:30565/TCP,15020:31820/TCP   4h

# service domain
kubectl get ksvc helloworld-go  --output=custom-columns=NAME:.metadata.name,DOMAIN:.status.domain
NAME            DOMAIN
helloworld-go   helloworld-go.default.example.com

# 访问测试
curl -H "Host: helloworld-go.default.example.com" http://35.192.111.18
Hello Go Sample v1!
```

## autoscrole-go
`github.com/istio/docs/docs/serving/samples/autoscale-go/service.yaml`
```yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: autoscale-go
  namespace: default
spec:
  runLatest:
    configuration:
      revisionTemplate:
        metadata:
          annotations:
            # Target 10 in-flight-requests per pod.
            autoscaling.knative.dev/target: "10"
        spec:
          container:
            image: gcr.io/knative-samples/autoscale-go:0.1
```

**部署**
```bash
kubectl apply -f https://raw.githubusercontent.com/knative/docs/master/docs/serving/samples/autoscale-go/service.yaml
```

**测试**
```bash
# service domain
kubectl get ksvc autoscale-go  --output=custom-columns=NAME:.metadata.name,DOMAIN:.status.domain
NAME           DOMAIN
autoscale-go   autoscale-go.default.example.com

# 访问测试
curl -H "Host: autoscale-go.default.example.com" "http://35.192.111.18?sleep=100&prime=10000&bloat=5"
Allocated 5 Mb of memory.
The largest prime less than 10000 is 9973.
Slept for 100.14 milliseconds.


# hey压测
hey -z 30s -c 50 \
  -host "autoscale-go.default.example.com" \
  "http://35.192.111.18?sleep=100&prime=10000&bloat=5"
  
# 或wrk压测
wrk -t 4 -d 30s -c 50 -H "Host: autoscale-go.default.example.com" "http://35.192.111.18?sleep=100&prime=10000&bloat=5"

# 可以多压几次或者时长、并发调大，看pods变化情况
kubectl get pods
NAME                                             READY     STATUS    RESTARTS   AGE
autoscale-go-jpmvw-deployment-54f494dd86-4ft4m   0/3       Pending   0          22s
autoscale-go-jpmvw-deployment-54f494dd86-j6x9j   3/3       Running   0          26s
autoscale-go-jpmvw-deployment-54f494dd86-ljqd7   3/3       Running   0          28s
autoscale-go-jpmvw-deployment-54f494dd86-s4xls   3/3       Running   0          28s
autoscale-go-jpmvw-deployment-54f494dd86-s767s   0/3       Pending   0          24s

```

| Key | Value |
|:---|:---|
|autoscaling.knative.dev/class|kpa.autoscaling.knative.dev<br/>hpa.autoscaling.knative.dev|
|autoscaling.knative.dev/metric|concurrency<br/>cpu|
|autoscaling.knative.dev/target|并发数/CPU占比|
|autoscaling.knative.dev/minScale|最小缩容Pod数|
|autoscaling.knative.dev/maxScale|最大扩容Pod数|

`annotations`定义
```go
const (
	InternalGroupName = "autoscaling.internal.knative.dev"

	GroupName = "autoscaling.knative.dev"

	// ClassAnnotationKey is the annotation for the explicit class of autoscaler
	// that a particular resource has opted into. For example,
	//   autoscaling.knative.dev/class: foo
	// This uses a different domain because unlike the resource, it is user-facing.
	ClassAnnotationKey = GroupName + "/class"
	// KPA is Knative Horizontal Pod Autoscaler
	KPA = "kpa.autoscaling.knative.dev"
	// HPA is Kubernetes Horizontal Pod Autoscaler
	HPA = "hpa.autoscaling.knative.dev"

	// MinScaleAnnotationKey is the annotation to specify the minimum number of Pods
	// the PodAutoscaler should provision. For example,
	//   autoscaling.knative.dev/minScale: "1"
	MinScaleAnnotationKey = GroupName + "/minScale"
	// MaxScaleAnnotationKey is the annotation to specify the maximum number of Pods
	// the PodAutoscaler should provision. For example,
	//   autoscaling.knative.dev/maxScale: "10"
	MaxScaleAnnotationKey = GroupName + "/maxScale"

	// MetricAnnotationKey is the annotation to specify what metric the PodAutoscaler
	// should be scaled on. For example,
	//   autoscaling.knative.dev/metric: cpu
	MetricAnnotationKey = GroupName + "/metric"
	// Concurrency is the number of requests in-flight at any given time.
	Concurrency = "concurrency"
	// CPU is the amount of the requested cpu actually being consumed by the Pod.
	CPU = "cpu"

	// TargetAnnotationKey is the annotation to specify what metric value the
	// PodAutoscaler should attempt to maintain. For example,
	//   autoscaling.knative.dev/metric: cpu
	//   autoscaling.knative.dev/target: 75   # target 75% cpu utilization
	TargetAnnotationKey = GroupName + "/target"

	// KPALabelKey is the label key attached to a K8s Service to hint to the KPA
	// which services/endpoints should trigger reconciles.
	KPALabelKey = GroupName + "/kpa"
)
```

## source-to-url-go

TODO