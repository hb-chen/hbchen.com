---
title: "【Tekton】组件介绍及安装部署"
date: 2021-03-28T00:41:51+08:00
draft: false
comments: true
mermaid: false
categories: [
"DevOps",
]
tags: [
"Tekton",
"CICD",
"Kubernetes",
]
---

Tekton 作为 Kubernetes 原生的 CI/CD 框架，拥有轻量、灵活等特点，接下来本文将介绍如何部署 Tekton 环境，后续以一个 Golang 项目为例，了解 Pipeline 和 Trigger，包括如何使用 Task、Pipeline、PipelineRun、EventListener、TriggerBinding、TriggerTemplate 等 CRDs 完成持续集成和交付的全流程。

<!--more-->

Tekton 包括 Pipeline、Trigger 和 Dashboard 三个部分，最小部署可以只用 Pipeline，Pipeline 是 Tekton 的核心，负责具体任务的流水线工作；Trigger 则主要是 Pipeline 的触发器，如有 Git PR 后通过 Webhook 触发 CI 流水线；Dashboard 则提供了一个 Web 控制台。要部署 Tekton 首先要有一个 K8S 环境，如果没有，可以使用 Minikube 做测试，如果有服务器资源推荐使用 [Rancher](https://rancher.com/)，可以选择部署一个 K3S 做 K8S 的多集群管理。

有了 K8S 环境 Tekton 的部署很简单，只需要`kubectl apply …`，但直接使用官方的`yaml`会有遇到一个问题GFW使`gcr.io`镜像无法访问，我在[tekton-practice/install](https://github.com/hb-chen/tekton-practice/tree/main/install)提供了国内镜像版本，如果可以科学上网，也可以使用[docker-sync](https://github.com/hb-chen/docker-sync)这个工具将镜像同步到自己的镜像仓库。

有了这些准备工作剩下部署很简单，以使用[tekton-practice](https://github.com/hb-chen/tekton-practice)为例：

```shell
git clone https://github.com/hb-chen/tekton-practice.git

cd tekton-practice

# 版本可以根据情况选择最新版本
kubectl apply -f install/pipeline_v0.20.0.yaml
kubectl apply -f install/trigger_v0.10.2.yaml
kubectl apply -f install/dashboard_v0.15.0.yaml
```

> 版本选择还要注意一点是三个组件间的兼容关系，可以参考[tektoncd/dashboard](https://github.com/tektoncd/dashboard#which-version-should-i-use)的 README 文档。

部署完成后可以查看下 K8S 的资源情况

```shell
# Pod
kubectl get po -n tekton-pipelines
NAME                                           READY   STATUS    RESTARTS   AGE
tekton-dashboard-5fcd8ddb84-qj6hr              1/1     Running   0          11d
tekton-pipelines-controller-7f7cfd5bdc-xb84j   1/1     Running   0          50d
tekton-pipelines-webhook-797666b499-dw6vp      1/1     Running   0          50d
tekton-triggers-controller-7b58bbd6db-6b6l2    1/1     Running   0          50d
tekton-triggers-webhook-59d767ddb-tk4jr        1/1     Running   0          50d

# Service
kubectl get svc -n tekton-pipelines
NAME                                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                              AGE
ingress-6a5bd2bac554b1f93e94a6160e460c0d   ClusterIP   10.43.55.190    <none>        9097/TCP                             50d
tekton-dashboard                           ClusterIP   10.43.134.48    <none>        9097/TCP                             43d
tekton-pipelines-controller                ClusterIP   10.43.211.12    <none>        9090/TCP,8080/TCP                    50d
tekton-pipelines-webhook                   ClusterIP   10.43.252.153   <none>        9090/TCP,8008/TCP,443/TCP,8080/TCP   50d
tekton-triggers-controller                 ClusterIP   10.43.3.54      <none>        9090/TCP                             50d
tekton-triggers-webhook                    ClusterIP   10.43.110.171   <none>        443/TCP                              50d
```

> Dashboard 本身没有安全认证，如果是暴露在公网的服务可以使用 nginx ingress 的 [basic-auth](https://kubernetes.github.io/ingress-nginx/examples/auth/basic/) 做一个简单的认证。

至此 Tekton 的基础部署完成，可以根据自身环境不同给 Dashboard 配置 Ingress 网关。下篇开始以 Golang 项目为例具体介绍如何使用 Pipeline 和 Trigger 完成持续集成与交付。
