# Issuer 配置

了解如何使用 Issuer 和 ClusterIssuer 源配置 cert-manager。

安装 cert-manager 后需要配置的第一件事是`Issuer` 或 `ClusterIssuer`。
这些源表示证书颁发机构(CAs)能够响应证书签名请求对证书进行签名。。

本节记录如何配置不同的颁发者类型。
您可能需要[阅读有关`Issuer` 和 `ClusterIssuer`源的更多信息](../concepts/issuer.md).

证书管理器带有许多内置的证书颁发者，这些证书颁发者在`cert-manager.io`组中表示。
除了内置类型之外，还可以安装外部颁发者。
内置和外部颁发者的待遇相同，配置也相似。

## 集群源命名空间

当使用`ClusterIssuer`源类型时，确保你理解集群源命名空间的用途;
对于刚开始使用 cert-manager 的人来说，这可能是一个常见的颁发源。

`ClusterIssuer`源是集群范围的。
这意味着当通过`secretName`字段引用一个秘密时，秘密将在`Cluster Resource Namespace`中查找。
默认情况下，这个命名空间是`cert-manager`，但是它可以通过 cert-manager-controller 组件上的一个标志来改变:

```bash
--cluster-resource-namespace=my-namespace
```
