---
title: "SelfSigned"
linkTitle: "自签名"
weight: 10
type: "docs"
---

该`SelfSigned`发行人并不代表证书颁发机构这样, 而是表示证书将通过“self signing”使用给定的私钥签名。.
这意味着产生的证书所提供的私有密钥将用来签署自己的证书.

这`Issuer`类型是非常有用 提供引导 CA 证书密钥对 对于一些私人密钥基础设施 (PKI), 或是自行创建简单证书.
客户消费这些证书都没有办法信任此证书 因为没有 CA 签名者除了本身, 正因为如此, 将被迫信任证书是.

> 注意: `CertificateRequests` 该引用自签名证书还必须包含注释 `cert-manager.io/private-key-secret-name`.
> 这是因为没有获得证书申请的私钥, 自签署证书的`CertificateRequest`将无法.
> 此注释由`Certificate`控制器自动添加.

## 部署

由于 `SelfSigned` `Issuer` 需要进行配置上的任何其他资源没有依赖性, 这是最简单的配置.
所需要的只是 对于`SelfSigned`节 存在于发行人`spec`.

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: selfsigned-issuer
  namespace: sandbox
spec:
  selfSigned: {}
```

可选, 您可以指定[CRL](https://en.wikipedia.org/wiki/Certificate_revocation_list) 分发点.
一个字符串数组其中每个标识从中该证书的撤销可以检查 CRL 的位置:

```
...
spec:
  selfSigned:
    crlDistributionPoints:
    - "http://example.com"
```

一旦部署完成，你应该能够立即看到发行人准备签约。
如果这是已经部署了`clusterissuers`替换`issuers`这里。

```bash
$ kubectl get issuers selfsigned-issuer -n sandbox -o wide
NAME                READY   STATUS                AGE
selfsigned-issuer   True                          2m
```

现在证书随时可以请求 通过使用在`sandbox`命名空间内名为 `selfsigned-issuer` 的 `SelfSigned` `Issuer`.
