---
title: Certificate
description: "cert-manager core concepts: Certificates"
---

# Certificate

cert-manager 有`Certificates`的概念，它定义了所需的 X.509 证书，该证书将被更新并保持最新。
一个`Certificates`是一个引用`Issuer` 或 `ClusterIssuer`的命名空间源，它们决定了什么将履行证书请求。

当创建`Certificates`时，相应的`CertificateRequest`源由 cert-manager 创建，其中包含编码的 X.509 证书请求、`Issuer`引用以及基于`Certificates`源规范的其他选项。

下面是一个`Certificate`源的例子。

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: acme-crt
spec:
  secretName: acme-crt-secret
  dnsNames:
    - example.com
    - foo.example.com
  issuerRef:
    name: letsencrypt-prod
    # We can reference ClusterIssuers by changing the kind here.
    # The default value is Issuer (i.e. a locally namespaced Issuer)
    kind: Issuer
    group: cert-manager.io
```

该`Certificate`将告诉 cert-manager 尝试使用名为`letsencrypt-prod`的`Issuer`来获取`example.com` 和 `foo.example.com`域的证书密钥对。
如果成功，生成的 TLS 密钥和证书将存储在名为`acme-crt-secret`的密钥中，密钥分别为`tls.key`和`tls.crt`。
这个秘密将位于与`Certificate`源相同的名称空间中。

当一个证书是由中间 CA 颁发的，并且`Issuer`可以提供颁发的证书链时，`tls.crt`的内容将是所请求的证书，后面是证书链。

此外，如果知道证书颁发机构，相应的 CA 证书将存储在密钥为`ca.crt`的秘密中。
例如，对于 ACME 颁发者，CA 是未知的，`ca.crt`将不存在于`acme-crt-secret`中。

cert-manager 有意避免向`tls.crt`添加根证书，因为它们在安全执行 TLS 的情况下是无用的。有关更多信息，请参见[RFC 5246 章节 7.4.2](https://datatracker.ietf.org/doc/html/rfc5246#section-7.4.2)，其中包含以下解释:

> 因为证书验证要求独立地分发根密钥，在假设远端必须已经拥有它才能在任何情况下验证它的前提下，指定根证书颁发机构的自签名证书可以从链中省略。

!!! Warning

    当配置客户端以使用由私有CA签名的服务证书连接到TLS服务器时，您将需要向客户端提供CA证书，以便其验证服务器。
    `ca.crt`可能包含您需要信任的证书，但 **不要挂载与服务器相同的`Secret`** 来访问`ca.crt`。

    这是因为:

    1. 这个`Secret`还包含服务器的私钥，应该只有服务器可以访问。
       您应该使用RBAC来确保包含服务证书和私钥的`Secret`只有需要它的Pods才能访问。
    1. 安全轮换CA证书依赖于能够同时信任旧CA证书和新CA证书。
       通过直接从源代码使用CA，这是不可能的;为了轮换证书，您将 *被迫* 有一些停机时间。

    在配置客户端时，您应该独立地选择并获取您想要信任的CA证书。
    从带外下载CA，并将其存储在与包含服务器私钥和证书的`Secret`分开的`Secret`或`ConfigMap`中。

    这确保了如果`Secret`中包含服务器密钥和证书的材料被篡改，
    客户端将无法连接到受威胁的服务器。

    同样的概念也适用于为相互验证的TLS配置服务器;
    不要让服务器访问包含客户端证书和私钥的`Secret`。

`dnsNames` 字段指定了一个与证书相关联的[`主题替代名称`](https://en.wikipedia.org/wiki/Subject_Alternative_Name)列表。

引用的`Issuer`必须与`Certificate`存在于相同的名称空间中。
`Certificate`也可以引用非命名空间的`ClusterIssuer`，因此可以从任何命名空间引用。

你可以阅读更多关于如何配置你的`Certificate`源[这里](../usage/certificate.md)。

## 证书生命周期

该图显示了使用 ACME / Let's Encrypt 颁发者的名为`cert-1`的证书的生命周期。
使用 cert-manager 不需要理解所有这些步骤;对于那些对这个过程好奇的人来说，这更多的是对发生在引擎盖下的逻辑的解释。

![证书的有效期](/images/letsencrypt-flow-cert-manager.png)
