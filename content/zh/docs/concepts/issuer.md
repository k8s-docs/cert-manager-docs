---
title: "发行者(Issuer)"
linkTitle: "发行者"
weight: 100
type: "docs"
description: >
  `Issuers`, 和 `ClusterIssuers`, 是 Kubernetes 资源 代表证书颁发机构 (CAs) 这能够通过声明证书签名请求生成签名证书.
---

所有 cert-manager 证书需要引用 Issuer 这是在就绪状态，试图接受该请求.

一个`Issuer`类型的一个例子是`CA`. 一个简单的`CA` `Issuer`如下:

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: ca-issuer
  namespace: mesh-system
spec:
  ca:
    secretName: ca-key-pair
```

这是一个简单的`Issuer` 将基于私钥签名证书.
存储在 secret `ca-key-pair`里证书然后，可以使用信任新签署的证书 通过这个在公共密钥基础设施 (PKI)里的`Issuer`
system.

## Namespaces

一个 `Issuer` 是一个名称空间资源, 并且不可在不同的命名空间里从 `Issuer` 能签发证书.
这表示 你需要在每个你想获得`Certificates`的命名空间创建一个`Issuer` .

如果你想创建一个可以在多个命名空间被消费的一个`Issuer`, 你应该考虑创建一个`ClusterIssuer`资源.
这几乎等同于`Issuer`资源, 然而是不被命名空间的,所以它可以在所有的命名空间被用来发出`Certificates` .

## 支持的 Issuers

cert-manager 支持多种 `in-tree`, 以及 `out-of-tree` `Issuer` 类型.
这些`Issuer`类型的详尽清单可以在证书管理器[配置文档](../../configuration/)中找到.
