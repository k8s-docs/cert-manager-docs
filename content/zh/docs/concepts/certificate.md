---
title: "证书(Certificate)"
linkTitle: "证书"
weight: 200
type: "docs"
---

cert-manager 有`Certificates`的概念限定期望的 X509 证书这将得到延续，并不断更新.
一个`Certificate`是一个名称空间资源引用了`Issuer`或`ClusterIssuer`其决定什么将履行证书请求.

当创建一个`Certificate`, 相应的`CertificateRequest`资源通过证书管理器创建包含编码 X509 证书请求, `Issuer` 参考, 和其他选项基于`Certificate`资源的规范.

这里是一个`Certificate`资源的这样一个例子.

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: acme-crt
spec:
  secretName: acme-crt-secret
  dnsNames:
    - foo.example.com
    - bar.example.com
  issuerRef:
    name: letsencrypt-prod
    # We can reference ClusterIssuers by changing the kind here.
    # The default value is Issuer (i.e. a locally namespaced Issuer)
    kind: Issuer
    group: cert-manager.io
```

这`Certificate`会告诉证书管理器尝试使用名为 ` letsencrypt-prod``Issuer ` 为 `foo.example.com` 和 `bar.example.com` 域获取证书密钥对 .
如果成功, 所得密钥和证书将被存储在一个名为 `acme-crt-secret` 分别带键 `tls.key`和`tls.crt`的 secret.
secret 将存活在`Certificate`资源相同的命名空间.

该`dnsNames`字段指定[`Subject Alternative Names`](https://en.wikipedia.org/wiki/Subject_Alternative_Name)列表要与证书相关联.

引用`Issuer`必须在同一个命名空间中的`Certificate`.
`Certificate`可以替代引用`ClusterIssuer` 其是非名称空间 所以可以从任何命名空间中引用.

你可以阅读更多关于如何配置你的`Certificate`资源[这里](../../usage/certificate/).
