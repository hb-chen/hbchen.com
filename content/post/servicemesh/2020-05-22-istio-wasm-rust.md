---
title: "【Istio】Rust 开发 wasm 扩展"
date: 2020-05-22
lastmod: 2020-05-24
draft: false
mermaid: false
categories: [
	"Service Mesh",
]
tags: [
	"Service Mesh",
    "Istio",
    "wasm",
    "envoy",
    "Rust"
]
---

Istio **1.5** 开始支持在数据面支持`Wasm`扩展，相关规范以及SDK在[proxy-wasm](https://github.com/proxy-wasm)，目前提供有三种 SDK 实现，分别是`C++`、`Rust`和`AssemblyScript`，其中`AssemblyScript`在[solo-io/proxy-runtime](https://github.com/solo-io/proxy-runtime)。除了 SDK Solo 还发布了 [WebAssembly Hub](https://webassemblyhub.io/)，并配有`wasme` CLI 工具，为 Wasm 扩展的开发提供了不错体验。本文主要介绍如何使用 Rust SDK 开发 Istio Wasm 扩展。

<!--more-->

## Wasme 工具

> [WebAssembly Hub Getting Started](https://docs.solo.io/web-assembly-hub/latest/tutorial_code/getting_started/)

```shell script
$ curl -sL https://run.solo.io/wasme/install | sh
$ export PATH=$HOME/.wasme/bin:$PATH
```

```shell script
$ wasme --version

wasme version 0.0.14
```

## Rust 项目

> [Rust安装](https://www.rust-lang.org/zh-CN/tools/install)
```shell script
$ rustc --version

rustc 1.43.1 (8d69840ab 2020-05-04)
```

创建Rust项目

使用`lib`模板创建项目
```shell script
$ cargo new hello-wold --lib 
```

编辑`Cargo.toml`

添加依赖
```toml
[dependencies]
log = "0.4.8"
proxy-wasm = "0.1.0" # The Rust SDK for proxy-wasm
```

配置动态库编译
```toml
[lib]
path = "src/lib.rs"
crate-type = ["cdylib"]
```

## Wasm 扩展

> 源码[hb-chen/wasm-hello-world-rust](https://github.com/hb-chen/wasm-hello-world-rust)

先通过 Rust SDK 了解扩展功能，编辑`src/lib.rs`
```rust
use proxy_wasm as wasm;

struct HelloWorld {
    context_id: u32,
}

impl wasm::traits::Context for HelloWorld {}
impl wasm::traits::RootContext for HelloWorld{}
impl wasm::traits::StreamContext for HelloWorld{}
impl wasm::traits::HttpContext for HelloWorld{}
```

安装依赖
```shell script
$ cargo build
```

要了解扩展看下几个 Context 的结构，其中`on_*`开头的便是扩展点，其他`get_*`、`set_*`、`add_*`等提供了不能接口能力。

为 Request 添加自定义 Header
```rust
use log::info;
use proxy_wasm as wasm;

#[no_mangle]
pub fn _start() {
    proxy_wasm::set_log_level(wasm::types::LogLevel::Trace);
    proxy_wasm::set_http_context(
        |context_id, _root_context_id| -> Box<dyn wasm::traits::HttpContext> {
            Box::new(HelloWorld { context_id })
        },
    )
}

struct HelloWorld {
    context_id: u32,
}

impl wasm::traits::Context for HelloWorld {}

impl wasm::traits::HttpContext for HelloWorld {
    fn on_http_request_headers(&mut self, num_headers: usize) -> wasm::types::Action {
        info!("Got {} HTTP headers in #{}.", num_headers, self.context_id);
        let headers = self.get_http_request_headers();
        let mut authority = "";

        for (name, value) in &headers {
            if name == ":authority" {
                authority = value;
            }
        }

        self.set_http_request_header("x-hello", Some(&format!("Hello world from {}", authority)));

        wasm::types::Action::Continue
    }
}
```

编译
```shell script
$ cargo build --target wasm32-unknown-unknown --release
```

`runtime-config.json`是`wasme build`时需要的，可以通过`wasme init`创建一个`cpp`项目获得
```json
{
  "type": "envoy_proxy",
  "abiVersions": [
    "v0-097b7f2e4cc1fb490cc1943d0d633655ac3c522f"
  ],
  "config": {
    "rootIds": [
      "hello_world"
    ]
  }
}
```

打包
```shell script
$ wasme build precompiled target/wasm32-unknown-unknown/release/hello_world.wasm --tag webassemblyhub.io/hbchen/hello_world:v0.1
```

```shell script
$ wasme list

NAME                          TAG  SIZE   SHA      UPDATED
webassemblyhub.io/hbchen/hello_world v0.1 1.9 MB 2e95e556 22 May 20 10:27 CST
```

> `wasme build`三种方式，
```shell script
$ wasme build -h

Available Commands:
  assemblyscript Build a wasm image from an AssemblyScript filter using NPM-in-Docker
  cpp            Build a wasm image from a CPP filter using Bazel-in-Docker
  precompiled    Build a wasm image from a Precompiled filter.
```

## 本地测试 Envoy

```shell script
$ wasme deploy envoy webassemblyhub.io/hbchen/hello_world:v0.1 --envoy-image=istio/proxyv2:1.6.0 --bootstrap=envoy-bootstrap.yml
```

```shell script
[2020-05-24 03:03:41.736][12][info][wasm] [external/envoy/source/extensions/common/wasm/context.cc:1103] wasm log hello_world hello_world : Got 16 HTTP headers in #9.
```

http://localhost:8080/headers

```json
{
  "headers" : {
    "X-Hello" : "Hello world from localhost:8080"
  }
}
```

> `wasme deploy`三种方式
```shell script
$ wasme deploy -h

Available Commands:
  envoy       Run Envoy locally in Docker and attach a WASM Filter.
  gloo        Deploy an Envoy WASM Filter to the Gloo Gateway Proxies (Envoy).
  istio       Deploy an Envoy WASM Filter to Istio Sidecar Proxies (Envoy).
```

## WebAssembly Hub 使用

> [Create a User on webassemblyhub.io & Log In from the wasme command line](https://docs.solo.io/web-assembly-hub/latest/tutorial_code/getting_started/#create-a-user-on-webassemblyhub-io-https-webassemblyhub-io)

Push 镜像
```shell script
$ wasme push webassemblyhub.io/hbchen/hello_world:v0.1
```

## 发布到 Istio

> httpbin 用例自己部署，本例`namespace=ns-httpbin`

可以分别通过`wasme`工具或`operator`添加扩展

### CLI 手动添加
```shell script
$ wasme deploy istio webassemblyhub.io/hbchen/hello_world:v0.1 \
  --id=hello-world-filter \
  --namespace=ns-httpbin
```

http://{ingress-host}:{port}/headers
```json
{
  "headers" : {
    "X-Hello" : "Hello world from {ingress-host}"
  }
}
```

卸载
```shell script
$ wasme undeploy istio \
  --id=hello-world-filter \
  --namespace=ns-httpbin
```

### Operator

CRD
```shell script
$ kubectl apply -f https://github.com/solo-io/wasme/releases/latest/download/wasme.io_v1_crds.yaml

customresourcedefinition.apiextensions.k8s.io/filterdeployments.wasme.io created
```


Operator
```shell script
$ kubectl apply -f https://github.com/solo-io/wasme/releases/latest/download/wasme-default.yaml

namespace/wasme created
configmap/wasme-cache created
serviceaccount/wasme-cache created
serviceaccount/wasme-operator created
clusterrole.rbac.authorization.k8s.io/wasme-operator created
clusterrole.rbac.authorization.k8s.io/wasme-cache created
clusterrolebinding.rbac.authorization.k8s.io/wasme-operator created
clusterrolebinding.rbac.authorization.k8s.io/wasme-cache created
daemonset.apps/wasme-cache created
deployment.apps/wasme-operator created
```

Filter
```yaml
$ kubectl apply -f - <<EOF
apiVersion: wasme.io/v1
kind: FilterDeployment
metadata:
  name: hello-world-filter
  namespace: ns-httpbin
spec:
  deployment:
    istio:
      kind: Deployment
  filter:
    image: webassemblyhub.io/hbchen/hello_world:v0.1
EOF
```

Filter 状态
```shell script
$ kubectl get filterdeployments.wasme.io -n ns-httpbin -o yaml hello-world-filter

status:
  observedGeneration: "1"
  workloads:
    httpbin:
      state: Succeeded
```

> 注意看下 Pod 是否成功，如果`READY 1/2`可能是`istio-proxy`没有启动，测试时这里遇到问题`wasme-cache`经常失败，可能是网络问题，并且失败后不能续传。可以进入 Pod 看缓存的尺寸是否和`hello_world.wasm`尺寸一致，缓存路径`/var/local/lib/wasme-cache/`，不一致就等缓存完成后再测试。
>

http://{ingress-host}:{port}/headers

```json
{
  "headers" : {
    "X-Hello" : "Hello world from {ingress-host}"
  }
}
```

卸载
```shell script
$ kubectl delete FilterDeployment -n ns-httpbin hello-world-filter
```

## Ref

- [WebAssembly Hub Tutorials](https://docs.solo.io/web-assembly-hub/latest/tutorial_code/)
- [Declarative WebAssembly deployment for Istio](https://istio.io/blog/2020/deploy-wasm-declarative/)
- [Extending Istio with Rust and WebAssembly](https://blog.red-badger.com/extending-istio-with-rust-and-webassembly)
- [Deploying Wasm Filters to Istio](https://docs.solo.io/web-assembly-hub/latest/tutorial_code/deploy_tutorials/deploying_with_istio/)
- [Extended and Improved WebAssemblyHub to Bring the Power of WebAssembly to Envoy and Istio](https://istio.io/blog/2020/wasmhub-istio/)
