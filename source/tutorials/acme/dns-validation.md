---
title: DNS 验证
description: "cert-manager turorials: 使用DNS验证颁发ACME证书"
---

# DNS 验证

## 使用 DNS 验证颁发 ACME 证书

cert-manager 可以通过[ACME](https://en.wikipedia.org/wiki/Automated_Certificate_Management_Environment)协议从 CA 获取证书。
ACME 协议支持各种挑战机制，这些机制用于证明域的所有权，以便为该域颁发有效的证书。

DNS01 就是这样一个挑战机制。通过 DNS01 挑战，您可以通过证明您控制其 DNS 记录来证明域的所有权。
这是通过创建具有特定内容的 TXT 记录来完成的，该记录证明您已经控制了域 DNS 记录。

以下颁发者定义了启用 DNS 验证所需的信息。
您可以在[Issuer docs](../../configuration/README.md)中阅读更多关于 Issuer 资源的信息。

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-staging
  namespace: default
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: user@example.com

    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-staging

    # ACME DNS-01 provider configurations
    solvers:
      # An empty 'selector' means that this solver matches all domains
      - selector: {}
        dns01:
          cloudDNS:
            # The ID of the GCP project
            # reference: https://cert-manager.io/docs/tutorials/acme/dns-validation/
            project: $PROJECT_ID
            # This is the secret used to access the service account
            serviceAccountSecretRef:
              name: clouddns-dns01-solver-svc-acct
              key: key.json

      # We only use cloudflare to solve challenges for example.org.
      # Alternative options such as 'matchLabels' and 'dnsZones' can be specified
      # as part of a solver's selector too.
      - selector:
          dnsNames:
            - example.org
        dns01:
          cloudflare:
            email: my-cloudflare-acc@example.com
            # !! Remember to create a k8s secret before
            # kubectl create secret generic cloudflare-api-key-secret
            apiKeySecretRef:
              name: cloudflare-api-key-secret
              key: api-key
```

我们已经为 Let’s Encrypt 的[登台环境](https://letsencrypt.org/docs/staging-environment/)指定了 ACME 服务器 URL。
登台环境不会颁发受信任的证书，但用于确保在转移到生产环境之前验证过程正常工作。
Encrypt 的生产环境施加了更严格的[速率限制](https://letsencrypt.org/docs/rate-limits/)，因此为了减少您触及这些限制的机会，强烈建议从使用登台环境开始。
要进入生产环境，只需将 URL 设置为`https://acme-v02.api.letsencrypt.org/directory`创建一个新的 Issuer。

ACME 协议的第一个阶段是客户端向 ACME 服务器注册。
此阶段包括生成一个非对称密钥对，然后将其与发行者中指定的电子邮件地址相关联。
请确保将此电子邮件地址更改为您拥有的有效电子邮件地址。
它通常用于在您的证书即将更新时发送到期通知。
生成的私钥存储在名为“letsencrypt-staging”的 Secret 中。

`dns01`节包含可用于解决 DNS 挑战的 DNS01 提供者列表。
我们的发行者定义了两个提供者。
这让我们可以在获取证书时选择使用哪一个。

有关 DNS 提供者配置的更多信息，包括受支持的提供者列表，可以在[DNS01 参考文档](../../configuration/acme/dns01/README.md)中找到。

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
  dnsNames:
    - "*.example.com"
    - example.com
    - example.org
```

Certificate 源描述了我们所需的证书以及可用于获取该证书的可能方法。
您可以像其他域一样获取通配符域的证书。
确保在 YAML 资源中用星号包装通配符域，以避免格式问题。
如果在同一个证书上同时指定`example.com` 和 `*.example.com`，执行验证所需的时间会稍微长一些，因为每个域都必须一个接一个地进行验证。
您可以在[docs](../../usage/README.md)中了解更多关于 Certificate 资源的信息。
如果成功获得证书，生成的密钥对将存储在名为`example-com-tls`的秘密中，与证书位于相同的名称空间中。

证书将有一个通用名称`*.example.com`，[主题替代名称(san)](https://en.wikipedia.org/wiki/Subject_Alternative_Name)将是`*.example.com`, `example.com` 和 `example.org`。

在我们的证书中，我们引用了上面的`letsencrypt-staging`颁发者。
颁发者必须与证书在相同的名称空间中。
如果你想引用一个`ClusterIssuer`，它是一个集群范围的 Issuer 版本，你必须在`issuerRef`节中添加`kind: ClusterIssuer`。

有关`ClusterIssuers`的更多信息，请阅读[issuer 概念](../../concepts/issuer.md).

`acme`节定义了 ACME 挑战的配置。
在这里，我们定义了用于验证域所有权的 DNS 挑战的配置。
对于`dns01`节中提到的每个域，cert-manager 将使用来自引用的颁发者的提供者凭据来创建一个名为`_acme-challenge`的 TXT 记录。
然后，ACME 服务器将对该记录进行验证，以便颁发证书。
一旦验证了域所有权，任何受证书管理器影响的记录都将被清除。

!!! Note

    您有责任确保所选择的提供者对您的域具有权威性。

在创建了上面的证书之后，我们可以使用`kubectl describe`检查它是否已经成功获得:

```bash
$ kubectl describe certificate example-com
Events:
  Type    Reason          Age      From          Message
  ----    ------          ----     ----          -------
  Normal  CreateOrder     57m      cert-manager  Created new ACME order, attempting validation...
  Normal  DomainVerified  55m      cert-manager  Domain "*.example.com" verified with "dns-01" validation
  Normal  DomainVerified  55m      cert-manager  Domain "example.com" verified with "dns-01" validation
  Normal  DomainVerified  55m      cert-manager  Domain "example.org" verified with "dns-01" validation
  Normal  IssueCert       55m      cert-manager  Issuing certificate...
  Normal  CertObtained    55m      cert-manager  Obtained certificate from ACME server
  Normal  CertIssued      55m      cert-manager  Certificate issued successfully
```

您还可以使用`kubectl get secret example-com-tls -o yaml`检查发布是否成功。
您应该看到一个 base64 编码的签名 TLS 密钥对。

获得证书后，证书管理器将定期检查其有效性，并在接近到期时尝试更新它。
当证书上的'Not After'字段小于当前时间加 30 天时，cert-manager 认为证书即将到期。
