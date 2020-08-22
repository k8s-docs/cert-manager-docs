---
title: "配置"
linkTitle: ""
weight: 30
type: "docs"
---

为了配置`CERT-manager`开始颁发证书, 首先 `Issuer` 或 `ClusterIssuer` 资源必须创建.
这些资源代表一个特定的签字权和细节证书请求将如何兑现.
你可以阅读更多关于`Issuers`的概念[这里](../concepts/issuer/).

`cert-manager` 支持多 'in-tree' 发行人类型 被记由`cert-manager.io`组中.
`cert-manager` 还支持外接发行相比，可以安装到您的集群属于其他组.
这些发行人的外部行为的类型没有什么不同，并在树发行人类型被视为等于.

当使用`ClusterIssuer`资源类型, 确保你理解了 [`群集资源命名空间`](../faq/cluster-resource/) 而其他 Kubernetes 资源将要参考.

## 支持发行人类型
