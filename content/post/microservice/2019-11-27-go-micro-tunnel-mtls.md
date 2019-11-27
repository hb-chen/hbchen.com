---
title: "【go-micro】Tunnel mTLS & 自签名证书"
date: 2019-11-27T20:39:14+08:00
lastmod: 2019-11-27 20:52:00
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

在[【go-micro】Network初探](/post/microservice/2019-11-15-go-micro-network/)我们分析了`network`的应用场景以及存在的不足之处，
其中对于安全不足研究的不够深入，`tunnel`是在`transport`基础上建立的，而`transport`层有`mTLS`的支持，所以当时所说的在安全方面只有`header`中的`token`是错误的。
只是`micro`当前在`network`还没有做`mTLS`环境变量的支持，本文将做一个简单的分享为`network`增加`mTLS`支持。

<!--more-->

> 刚刚(2019-11-27)`micro`又发布了新版本`1.17.0`，继续采坑🤣

## Network修改
当前`micro`的`network`模块并没有提供`TLS`环境变量配置的支持，需要自己修改源码，在[micro/micro](https://github.com/micro/micro)的`internal/helper`有`TLSConfig()`方法可以从`GLOBAL OPTIONS`中生成`*tls.Config`，
参考当前`micro`实现`mTLS`两个`command`(`api`和`web`)，对`network`的`tunnel`做如下修改:
```go
// create a tunnel
tunOpts := []tunnel.Option{
	tunnel.Address(Address),
	tunnel.Nodes(nodes...),
	tunnel.Token(Token),
}
if ctx.GlobalBool("enable_tls") {
	config, err := helper.TLSConfig(ctx)
	if err != nil {
		fmt.Println(err.Error())
		return
	}
	config.InsecureSkipVerify = true

	tunOpts = append(tunOpts, tunnel.Transport(
		quic.NewTransport(transport.TLSConfig(config)),
	))
}
tun := tunnel.NewTunnel(tunOpts...)
```

> 注意这里我们设置了`InsecureSkipVerify`为`true`，由于是双向认证，如果`InsecureSkipVerify`为`false`，`network`将无法正常连接

## mTLS
首先参考下图对`mTLS`有个了解，`server`和`client`都需要通过`CA`对彼此的证书进行验证

![network_multi_cluster](/img/micro/mtls.png)

我们只生成一个`CA`和`CSR`证书，对于生产环境根据自己的场景需要更为完善和复杂的证书管理，这里没有涉及，后面会结合`micro`生态与安全相关的`mTLS`做更多的研究分享。

### CA证书
使用[certstrap](https://github.com/square/certstrap)工具生成`CA`证书
```bash
# MacOS
$ brew install certstrap

# 未输入密码
$ certstrap init --common-name "MicroCA" --expires "20 years"
    Enter passphrase (empty for no passphrase): 
    Enter same passphrase again: 
    Created out/MicroCA.key
    Created out/MicroCA.crt
    Created out/MicroCA.crl
```

### CSR证书
```bash
$ certstrap request-cert -cn network  
	Enter passphrase (empty for no passphrase): 
    Enter same passphrase again: 
    Created out/network.key
    Created out/network.csr
```

### CSR证书签名
```bash
$ certstrap sign network --CA MicroCA
	Created out/network.crt from out/network.csr signed by out/MicroCA.key
```

## 运行Network + mTLS

```bash
$ ./micro \
--registry=etcd \
--transport=tcp \
--enable_tls=true \
--tls_cert_file=conf/tls/network.crt  \
--tls_key_file=conf/tls/network.key  \
--tls_client_ca_file=conf/tls/MicroCA.crt \
network

$ ./micro \
--registry=consul \
--transport=tcp \
--enable_tls=true \
--tls_cert_file=conf/tls/network.crt  \
--tls_key_file=conf/tls/network.key  \
--tls_client_ca_file=conf/tls/MicroCA.crt \
network \
--nodes=localhost:8085 \
--address=:8086
```

**Network routes**

> 如果`routes`中看不到`link=network`应该是`network`间未能建立连接

```bash
./micro --registry=etcd --transport=tcp network routes  
+------------------+----------------------+----------------------+--------------------------------------+----------+--------+---------+
|     SERVICE      |       ADDRESS        |       GATEWAY        |                ROUTER                | NETWORK  | METRIC |  LINK   |
+------------------+----------------------+----------------------+--------------------------------------+----------+--------+---------+
| consul           | 1919625842587659968  | 17017708900476934200 | f5dc3933-3ccc-4dc0-bafe-cbfd7abebf60 | go.micro | 1      | network |
| go.micro.network | 15624894091238291400 | 17017708900476934200 | f5dc3933-3ccc-4dc0-bafe-cbfd7abebf60 | go.micro | 1      | network |
| go.micro.network | 192.168.1.4:58527    |                      | df521f3c-a39e-455b-abbf-ada184a900c9 | go.micro | 1      | local   |
| go.micro.network | 192.168.1.4:58528    |                      | df521f3c-a39e-455b-abbf-ada184a900c9 | go.micro | 1      | local   |
| go.micro         | 192.168.1.4:8085     |                      | df521f3c-a39e-455b-abbf-ada184a900c9 | go.micro | 1      | local   |
| go.micro         | 9876822083478954444  | 17017708900476934200 | f5dc3933-3ccc-4dc0-bafe-cbfd7abebf60 | go.micro | 1      | network |
+------------------+----------------------+----------------------+--------------------------------------+----------+--------+---------+

./micro --registry=consul --transport=tcp network routes
+------------------+----------------------+---------------------+--------------------------------------+----------+--------+---------+
|     SERVICE      |       ADDRESS        |       GATEWAY       |                ROUTER                | NETWORK  | METRIC |  LINK   |
+------------------+----------------------+---------------------+--------------------------------------+----------+--------+---------+
| consul           | 127.0.0.1:8300       |                     | f5dc3933-3ccc-4dc0-bafe-cbfd7abebf60 | go.micro | 1      | local   |
| go.micro         | 192.168.1.4:8086     |                     | f5dc3933-3ccc-4dc0-bafe-cbfd7abebf60 | go.micro | 1      | local   |
| go.micro         | 9480410441638176179  | 3307701226171868606 | df521f3c-a39e-455b-abbf-ada184a900c9 | go.micro | 2      | network |
| go.micro.network | 11801771601773192119 | 3307701226171868606 | df521f3c-a39e-455b-abbf-ada184a900c9 | go.micro | 2      | network |
| go.micro.network | 192.168.1.4:58843    |                     | f5dc3933-3ccc-4dc0-bafe-cbfd7abebf60 | go.micro | 1      | local   |
| go.micro.network | 192.168.1.4:58844    |                     | f5dc3933-3ccc-4dc0-bafe-cbfd7abebf60 | go.micro | 1      | local   |
+------------------+----------------------+---------------------+--------------------------------------+----------+--------+---------+
```

## Ref

[Go 编程: 快速生成自签名证书与双向认证(mTLS)](https://www.gitdig.com/about/)
