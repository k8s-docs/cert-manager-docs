---
title: HTTP Validation
description: "cert-manager tutorials: Issuing an ACME certificate using HTTP validation"
---

# HTTP 验证

## 使用 HTTP 验证颁发 ACME 证书

cert-manager 可以通过[ACME](https://en.wikipedia.org/wiki/Automated_Certificate_Management_Environment)协议从 CA 获取证书。
ACME 协议支持各种挑战机制，这些机制用于证明域的所有权，以便为该域颁发有效的证书。

其中一个这样的挑战机制是 HTTP01 挑战。
使用 HTTP01 挑战，您可以通过确保域中存在特定文件来证明域的所有权。
如果您能够在给定路径下发布给定文件，则假定您控制了域。

下面的发布者定义了启用 HTTP 验证所需的信息。
您可以在[Issuer docs](../../concepts/issuer.md)中阅读更多关于 Issuer 资源的信息。

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-staging
  namespace: default
spec:
  acme:
    # The ACME server URL
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: user@example.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-staging
    # Enable the HTTP-01 challenge provider
    solvers:
      # An empty 'selector' means that this solver matches all domains
      - selector: {}
        http01:
          ingress:
            class: nginx
```

我们已经为 Let’s Encrypt 的[登台环境](https://letsencrypt.org/docs/staging-environment/)指定了 ACME 服务器 URL。
登台环境不会颁发受信任的证书，但用于确保在转移到生产环境之前验证过程正常工作。
让我们加密的生产环境强加更严格的[速率限制](https://letsencrypt.org/docs/rate-limits/)，所以为了减少您达到这些限制的机会，强烈建议从使用登台环境开始。
要进入生产环境，只需将 URL 设置为`https://acme-v02.api.letsencrypt.org/directory`创建一个新的 Issuer。

ACME 协议的第一个阶段是客户端向 ACME 服务器注册。
此阶段包括生成一个非对称密钥对，然后将其与发行者中指定的电子邮件地址相关联。
请确保将此电子邮件地址更改为您拥有的有效电子邮件地址。
它通常用于在您的证书即将更新时发送到期通知。
生成的私钥存储在名为`letsencrypt-staging`的 Secret 中。

我们必须提供一个或多个解算器来处理 ACME 挑战。
在这种情况下，我们想要使用 HTTP 验证，所以我们指定了一个`http01`求解器。
我们可以选择映射不同的域来使用不同的 Solver 配置。

一旦我们创建了上面的颁发者，我们就可以使用它来获取证书。

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com
  namespace: default
spec:
  secretName: example-com-tls
  issuerRef:
    name: letsencrypt-staging
  commonName: example.com
  dnsNames:
    - www.example.com
```

Certificate 源描述了我们所需的证书以及可用于获取该证书的可能方法。
您可以在[docs](../../concepts/certificate.md)中了解更多关于 Certificate 资源的信息。
如果成功获得证书，生成的密钥对将存储在名为`example-com-tls`的秘密中，与证书位于相同的名称空间中。

证书将有一个通用名称`example.com`，[主题替代名称(SANs)](https://en.wikipedia.org/wiki/Subject_Alternative_Name) 将是`example.com`和`www.example.com`。
请注意，TLS 客户端只支持这些 SANs。

在我们的证书中，我们引用了上面的`letsencrypt-staging`颁发者。
颁发者必须与证书在相同的名称空间中。
如果你想引用一个`ClusterIssuer`，它是一个集群范围的 Issuer 版本，你必须在`issuerRef` 节中添加`kind: ClusterIssuer`。

有关`ClusterIssuers`的更多信息，请阅读[`ClusterIssuers` 文档](../../concepts/issuer.md).

`acme`节定义了 ACME 挑战的配置。
在这里，我们已经定义了用于验证域所有权的 HTTP01 挑战的配置。
为了验证`http01`节中提到的每个域的所有权，cert-manager 将创建一个 Pod、Service 和 Ingress，以公开一个满足 HTTP01 挑战的 HTTP 端点。

`http01` 节中的`ingress` 和 `ingressClass`字段可以用来控制 cert-manager 如何与 ingress 资源交互:

- 如果指定了`ingress`字段，则必须已经存在与 Certificate 在同一名称空间中的同名 ingress 资源，并且只会修改它以添加适当的规则来解决挑战。
  该字段对于谷歌 Cloud Loadbalancer 入口控制器以及许多其他为每个入口资源分配单个公共 IP 地址的控制器非常有用。
  如果没有人工干预，创建新的入口资源将导致任何挑战失败。
- 如果指定了`ingressClass`字段，则将创建一个具有随机生成名称的新入口资源以解决该挑战。
  这个新资源将有一个带有`kubernetes.io/ingress.class`键的注释，值设置为`ingressClass`字段的值。
  这适用于 NGINX 入口控制器。
- 如果两者都没有指定，则将使用随机生成的名称创建新的入口资源，但它们将没有入口类注释集。
- 如果两者都指定了，那么`ingress`字段将优先。

一旦验证了域所有权，任何受证书管理器影响的资源都将被清除或删除。

!!! Note

    将每个域名指向您的入口控制器的正确IP地址是您的责任。

在创建了上面的证书之后，我们可以使用`kubectl describe`检查它是否已经成功获得:

```bash
$ kubectl describe certificate example-com
Events:
  Type    Reason          Age      From          Message
  ----    ------          ----     ----          -------
  Normal  CreateOrder     57m      cert-manager  Created new ACME order, attempting validation...
  Normal  DomainVerified  55m      cert-manager  Domain "example.com" verified with "http-01" validation
  Normal  DomainVerified  55m      cert-manager  Domain "www.example.com" verified with "http-01" validation
  Normal  IssueCert       55m      cert-manager  Issuing certificate...
  Normal  CertObtained    55m      cert-manager  Obtained certificate from ACME server
  Normal  CertIssued      55m      cert-manager  Certificate issued successfully
```

您还可以使用`kubectl get secret example-com-tls -o yaml`检查发布是否成功。
您应该看到一个 base64 编码的签名 TLS 密钥对。

获得证书后，证书管理器将定期检查其有效性，并在接近到期时尝试更新它。
当证书上的 'Not After' 字段小于当前时间加 30 天时，cert-manager 认为证书即将到期。
