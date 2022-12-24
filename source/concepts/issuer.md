---
title: Issuer(颁发者)
description: "cert-manager核心概念:issuer和clusterissuer"
---

# Issuer(颁发者)

`Issuers`, 和 `ClusterIssuers`,是 Kubernetes 源，代表证书颁发机构(CAs)，能够通过执行证书签名请求来生成已签名的证书。
所有证书管理器证书都需要一个处于准备状态的被引用颁发者来尝试履行请求。

`Issuer`类型的一个例子是`CA`。简单的`CA` `Issuer`如下:

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: ca-issuer
  namespace: mesh-system
spec:
  ca:
    secretName: ca-key-pair
```

这是一个简单的`Issuer`，将根据私钥签署证书。
然后，存储在秘密`ca-key-pair`中的证书可用于信任公共密钥基础设施(PKI)系统中该`Issuer`新签署的证书。

## 命名空间

`Issuer`是一个名称空间源，不可能从不同名称空间中的`Issuer`颁发证书。
这意味着您需要在每个希望获取“证书”的名称空间中创建一个`Issuer`。

如果您希望创建一个可以在多个名称空间中使用的`Issuer`，则应该考虑创建一个`ClusterIssuer`源。
这与`Issuer`源几乎相同，但是它是非命名空间的，因此可以用于跨所有命名空间颁发`Certificates`。

## 支持颁发者

cert-manager 支持许多`in-tree`和`out-of-tree`的`Issuer`类型。
这些`Issuer`类型的详尽列表可以在 cert-manager[配置文档](../configuration/README.md)中找到。
