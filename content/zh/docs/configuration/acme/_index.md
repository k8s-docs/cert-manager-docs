---
title: "Automated Certificate Management Environment(ACME)"
linkTitle: "ACME"
weight: 60
type: "docs"
---

在 `ACME` `Issuer` 型代表单个帐户 用自动证书管理环境注册 (ACME) 证书颁发机构服务器.
当你创建一个新的 `ACME` `Issuer`, cert-manager 将生成私钥 这是用来识别您的身份连接`ACME`服务器.

公共`ACME`服务器颁发的证书通常是由默认客户端的计算机信任.
这意味着，例如, 访问由该 URL 发出`ACME`证书支持的网站, 在默认情况下，大多数客户端的 Web 浏览器信任.
`ACME` 证书通常免费.

## 解决 challenges

为了在`ACME` CA 服务器来验证客户端拥有该域名, 或域, 被请求的证书是为, 客户必须填写 "challenges".
这是为了确保客户端将无法对域请求证书 他们没有自己，结果, 欺诈冒充他人的网站.
如在[RFC8555](https://tools.ietf.org/html/rfc8555)中详述, cert-manager 提供了两个 challenges 验证 - HTTP01 和 DNS01 challenges.

[HTTP01](./http01/) challenges 完成 通过呈现一个计算键值, 这应该是出现在一个 HTTP URL 端点 并通过互联网路由.
此 URL 将使用申请证书的域名.
一旦`ACME`服务器能够在互联网上得到这个 URL 此键, 在 ACME 服务器可以验证您是该域名的所有者.
当创建一个 HTTP01 挑战, cert-manager 将自动配置群集入口将流量路由此网址一个小的 Web 服务器呈现这一键.

[DNS01](./dns01/) challenges 完成 通过提供一个存在于 DNS TXT 记录里计算的密钥.
一旦这个 TXT 记录已经在互联网上传播, 在`ACME`服务器可以成功检索通过 DNS 查找此键，可以验证客户端拥有对所请求证书的站点.
有了正确的权限, cert-manager 会自动出现该 TXT 记录了给定的 DNS 提供商.

## 配置

### 创建基本`ACME`Issuer

所有 `ACME` `Issuers` 遵循类似的配置结构 - 一个客户 `email`,
一个 `server` 网址, 一个 `privateKeySecretRef`, 和一个或多个 `solvers`.
下面是一个简单的例子`ACME`发行者:

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # You must replace this email address with your own.
    # Let's Encrypt will use this to contact you about expiring
    # certificates, and issues related to your account.
    email: user@example.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Secret resource that will be used to store the account's private key.
      name: example-issuer-account-key
    # Add a single challenge solver, HTTP01 using nginx
    solvers:
      - http01:
          ingress:
            class: nginx
```

解算器进来的[`dns01`](./dns01/) 和 [`http01`](./http01/)的形式 节.
有关如何配置这些求解器类型的详细信息, 访问他们各自的文件 - [DNS01](./dns01/), [HTTP01](./http01/).

### 外部帐户绑定

使用外部帐户绑定你的`ACME`帐户 cert-Manager 支持.
外部帐户绑定用于你的与外部帐户`ACME`帐户关联 如 CA 自定义数据库.
这通常是没有必要对大多数证书管理器的用户，除非你知道这是需要明确.

外部帐户绑定在`ACME` `Issuer`上需要三个字段 这代表你的`ACME`帐户.
这些字段; 其中`keyID`您的外部账号绑定由外部客户经理指数, `keySecretRef`它引用一个秘密包含您的外部帐户对称 MAC 密钥的基 64 编码的 URL 字符串, 最后 `keyAlgorithm`, MAC 算法用来签署包含您的外部账号注册与`ACME`服务器帐户时绑定 JSON 字符串网.

> 注意: 如果它是不是已经命令`base64`是编码的 MAC 密钥有用 `$ echo 'my-secret-key' | base64`,
> 那么你可以创建一个秘密的资源: `kubectl create secret generic eab-secret --from-literal secret={base64 encoded secret key}`

一个`ACME`发行者与外部帐户结合的一个例子如下。

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: my-acme-server-with-eab
spec:
  acme:
    email: user@example.com
    server: https://my-acme-server-with-eab.com/directory
    externalAccountBinding:
      keyID: my-kid-1
      keySecretRef:
        name: eab-secret
        key: secret
      keyAlgorithm: HS256
    privateKeySecretRef:
      name: example-issuer-account-key
    solvers:
      - http01:
          ingress:
            class: nginx
```

### 添加多个求解类型

你可能想，如果你想使用`DNS01`旁边正在使用`HTTP01`验证的其他证书颁发通配符证书使用不同类型的挑战求解器配置为不同的入口控制器，例如。

该`solvers`节有一个可选的`selector`字段, 可用于指定哪些`Certificates`, 并进一步, 什么 DNS 这些`Certificates`名称应该被用来解决难题.

存在可以被用于形成要求三个选择类型的一个`Certificate`必须满足被选择用于一个解算器 - `matchLabels`，`dnsNames`和`dnsZones`. 你可以在一个单一的求解任意数量的这三种选择的.

#### 匹配标签

该`matchLabel`选择要求所有`Certificates`比赛在节中的字符串映射列表中定义的标签的至少一个。
例如, 下面`Issuer`将只匹配`Certificates` 有标签 `"user-cloudflare-solver": "true"`, 要么 `"email": "user@example.com"`, 要么 都.

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    ...
    solvers:
    - dns01:
        cloudflare:
          email: user@example.com
          apiKeySecretRef:
            name: cloudflare-apikey-secret
            key: apikey
      selector:
        matchLabels:
          "use-cloudflare-solver": "true"
          "email": "user@example.com"
```

#### DNS 名称

该`dnsNames`选择是应该被映射到一个确切的解算器的 DNS 名称的列表。
这意味着，`包含任意这些DNS名称的Certificates`将被选中。
如果发现匹配, 一个`dnsNames`选择器将优先于[`dnsZones`](#dns-zones)选择.
如果有多个解算器匹配相同`dnsNames`值, 与[`matchLabels`](#match-labels)将选择最匹配的标签，求解器.
如果既没有更多的比赛，前面定义的列表将被选择的解算器。

下面的例子将解决 Certificates`的`挑战 与 DNS 名称 `example.com` 和 `*.example.com` 这些域.

> 注意: `dnsNames`采取完全匹配，不解决通配符,这意味着下面`Issuer`不会解决 DNS 名称，如 `foo.example.com`.
> 使用 [`dnsZones`](#dns-zones) selector 类型匹配一个区域内的所有子域.

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    ...
    solvers:
    - dns01:
        cloudflare:
          email: user@example.com
          apiKeySecretRef:
            name: cloudflare-apikey-secret
            key: apikey
      selector:
        dnsNames:
        - 'example.com'
        - '*.example.com'
```

#### DNS 区

该`dnsZones`节定义了可以通过该解算器来解决 DNS 区域的列表。
如果 DNS 名称是完全匹配, 或任何指定的`dnsZones`的子域, 该解算器将使用, 除非更具体的[`dnsNames`](#dns-names)匹配被配置.
这意味着，`sys.example.com`将在一个被选择指定`example.com`域`www.sys.example.com`。
如果有多个解算器匹配相同`dnsZones`值, 与[`matchLabels`](#match-labels)将选择最匹配的标签，求解器.
如果既没有更多的比赛，前面定义的列表将被选择的解算器。

在下面的例子, 该解算器将解决域`example.com`挑战, 以及所有及其子域的`* .example.com`.

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    ...
    solvers:
    - dns01:
        cloudflare:
          email: user@example.com
          apiKeySecretRef:
            name: cloudflare-apikey-secret
            key: apikey
      selector:
        dnsZones:
        - 'example.com'
```

#### 全部一起

每个解算器能够有任意数量的定义的三种选择类型。
在下面的例子, 在`DNS01`求解器将用于解决难题为`Certificates`域 包含 DNS 名称`a.example.com`和`b.example.com`, 或 `test.example.com` 和所有及其子域的 (例如 `foo.test.example.com`).

对于所有其他的挑战, 在`HTTP01`求解器将只用于当`Certificate`还包含标签 `"use-http01-solver": "true"`.

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    ...
    solvers:
    - http01:
        ingress:
          class: nginx
      selector:
        matchLabels:
          "use-http01-solver": "true"
    - dns01:
        cloudflare:
          email: user@example.com
          apiKeySecretRef:
            name: cloudflare-apikey-secret
            key: apikey
      selector:
        dnsNames:
        - 'a.example.com'
        - 'b.example.com'
        dnsZones:
        - 'test.example.com'
```
