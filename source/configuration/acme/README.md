# ACME

ACME 颁发者类型表示在自动证书管理环境(ACME)证书颁发机构服务器上注册的单个帐户。
当您创建一个新的 ACME `Issuer`时，证书管理器将生成一个用于在 ACME 服务器上识别您的私钥。

默认情况下，客户端计算机通常信任公共 ACME 服务器颁发的证书。
这意味着，例如，访问一个由为该 URL 颁发的 ACME 证书支持的网站，将在默认情况下被大多数客户端的 web 浏览器信任。
ACME 证书通常是免费的。

## 解决挑战

为了让 ACME CA 服务器验证客户机是否拥有一个或多个域，正在请求证书，客户机必须完成“质询”。
这是为了确保客户端无法为他们不拥有的域名请求证书，从而欺诈性地模仿其他人的网站。
详见[RFC8555](https://tools.ietf.org/html/rfc8555)，cert-manager 提供了两个挑战验证- HTTP01 和 DNS01 挑战。

[HTTP01](./http01/README.md)挑战是通过给出一个计算密钥来完成的，该密钥应该出现在 HTTP URL 端点上，并且可以在互联网上路由。
该 URL 将使用证书请求的域名。
一旦 ACME 服务器能够通过 internet 从这个 URL 获得这个密钥，ACME 服务器就可以验证您是这个域的所有者。
当创建 HTTP01 挑战时，cert-manager 将自动配置您的集群入口，将此 URL 的流量路由到提供此密钥的小型 web 服务器。

[DNS01](./dns01/README.md)挑战通过提供一个存在于 DNS TXT 记录中的计算密钥来完成。
一旦这个 TXT 记录在互联网上传播，ACME 服务器就可以通过 DNS 查找成功地检索这个密钥，并验证客户端拥有所请求证书的域。
有了正确的权限，证书管理器将自动为给定的 DNS 提供程序显示此 TXT 记录。

## 配置

### 创建基本 ACME 颁发者

所有 ACME `Issuers`都遵循类似的配置结构 - 一个客户`email`，一个`server`URL，一个`privateKeySecretRef`，以及一个或多个`solvers`。
下面是一个简单的 ACME 颁发者示例:

```yaml
apiVersion: cert-manager.io/v1
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

求解器以[`dns01`](./dns01/README.md) 和 [`http01`](./http01/README.md)节的形式出现。
有关如何配置这些求解器类型的更多信息，请访问它们各自的文档- [DNS01](./dns01/README.md), [HTTP01](./http01/README.md)。

### 外部帐户绑定

cert-manager 支持对您的 ACME 帐户使用外部帐户绑定。
外部帐户绑定用于将您的 ACME 帐户与外部帐户(如 CA 自定义数据库)相关联。
大多数 cert-manager 用户通常不需要这样做，除非您知道明确需要这样做。

外部帐户绑定需要在代表您的 ACME 帐户的 ACME“发行者”上设置两个字段。这些字段是:

- `keyID` - 外部帐户管理员索引您的外部帐户绑定的密钥 ID 或帐户 ID
- `keySecretRef` - 一个秘密的名称和密钥，包含一个 base 64 编码的 URL 字符串的外部帐户对称 MAC 密钥

!!! Note:

    在大多数情况下，MAC密钥必须编码为`base64URL`。
    下面的命令将对一个键进行base64-encode，并将其转换为`base64URL`:

    ```console
    $ echo 'my-secret-key' | base64 -w0 | sed -e 's/+/-/g' -e 's/\//_/g' -e 's/=//g'
    ```

    然后，您可以创建Secret资源:

    ```console
    $ kubectl create secret generic eab-secret --from-literal \
      secret={base64 encoded secret key}
    ```

具有外部帐户绑定的 ACME 颁发者示例如下。

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: my-acme-server-with-eab
spec:
  acme:
    email: user@example.com
    server: https://my-acme-server-with-eab.com/directory
    externalAccountBinding:
      keyID: my-keyID-1
      keySecretRef:
        name: eab-secret
        key: secret
    privateKeySecretRef:
      name: example-issuer-account-key
    solvers:
      - http01:
          ingress:
            class: nginx
```

!!! Note

    `v1.3.0`之前的cert-manager版本也要求用户通过设置`Issuer.spec.acme.externalAccountBinding.keyAlgorithm`字段指定EAB的MAC算法。
    该字段现在已弃用，因为上游Go `x/crypto`库将算法硬编码为`HS256`。
    (参见上游相关讨论[`CL#41430`](https://github.com/golang/go/issues/41430))。

### 重用 ACME 帐户

您可能希望跨多个集群重用单个 ACME 帐户。
这在使用 EAB 时尤其有用。
如果设置了`disableAccountKeyGeneration`字段，证书管理器将不会创建一个新的 ACME 帐户，并使用`privateKeySecretRef`中指定的现有密钥。
请注意，`Issuer`/`ClusterIssuer`将不会准备好，并将继续重试，直到`Secret`被提供。

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: my-acme-server-with-existing-acme-account
spec:
  acme:
    email: user@example.com
    disableAccountKeyGeneration: true
    privateKeySecretRef:
      name: example-issuer-account-key
```

### 添加多个求解器类型

您可能希望为不同的入口控制器使用不同类型的挑战解决器配置，例如，如果您想使用`DNS01`颁发通配符证书，同时使用`HTTP01`验证其他证书。

`solvers`节有一个可选的`selector`字段，可以用来指定哪些`Certificates`，以及这些`Certificates`上的哪些 DNS 名称应该用于解决挑战。

有三种选择器类型可以用来形成`Certificates`必须满足的要求，以便被选择为求解器-`matchLabels`,`dnsNames` and `dnsZones`。
你可以在一个求解器中使用任意数量的这三个选择器。

#### 匹配标签

`matchLabel`选择器要求所有`Certificates`匹配该节字符串映射列表中定义的所有标签。
例如，下面的`Issuer`只会匹配标签为`"user-cloudflare-solver": "true"` 和 `"email": "user@example.com"`。

```yaml
apiVersion: cert-manager.io/v1
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

#### DNS 的名字

`dnsNames`选择器是一个精确的 DNS 名称列表，应该映射到一个求解器。
这意味着包含任何这些 DNS 名称的“证书”将被选中。
如果找到匹配，`dnsNames`选择器将优先于 [`dnsZones`](#dns-zones) 选择器。
如果多个解算器匹配相同的`dnsNames`值，则在 [`matchLabels`](#match-labels)中匹配标签最多的解算器将被选中。
如果两者都没有更多匹配项，则将选择列表中前面定义的求解器。

下面的示例将解决这些域的 DNS 名称为`example.com` and `*.example.com`的`Certificates`挑战。

!!! Note

    `dnsNames`采用精确匹配，不解析通配符，这意味着下面的`Issuer`将无法解决诸如`foo.example.com`之类的DNS名称。
    使用[`dnsZones`](#dns-zones) 选择器类型来匹配区域内的所有子域。

```yaml
apiVersion: cert-manager.io/v1
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

`dnsZones`节定义了此求解器可以解决的 DNS 区域列表。
如果一个 DNS 名称是精确匹配的，或者是任何指定的`dnsZones`的子域，这个求解器将被使用，除非配置了更具体的 [`dnsNames`](#dns-names) 匹配。
这意味着`sys.example.com`将被选中，而不是为域名`www.sys.example.com`指定`example.com`。
如果多个解算器匹配相同的`dnsZones`值，将选择[`matchLabels`](#match-labels)中匹配标签最多的解算器。
如果两者都没有更多匹配项，则将选择列表中前面定义的求解器。

在下面的例子中，这个求解器将解决域`example.com`及其所有子域`*.example.com`的挑战。

```yaml
apiVersion: cert-manager.io/v1
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

#### 在一起

每个求解器都可以定义这三种选择器类型中的任意数量。
在以下示例中，CloudFlare 的`DNS01`求解器将用于解决包含 DNS 名称`a.example.com` 和 `b.example.com`的`Certificates` 域的挑战。
谷歌 CloudDNS 的`DNS01`求解器将用于解决`Certificates` 的挑战，其 DNS 名称匹配区域`test.example.com` 及其所有子域(例如`foo.test.example.com`)。

对于所有其他挑战，只有当`Certificate`还包含`"use-http01-solver": "true"`标签时，才会使用`HTTP01`求解器。

```yaml
apiVersion: cert-manager.io/v1
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
    - dns01:
      cloudDNS:
        project: my-project-id
        hostedZoneName: 'test-example.com'
        serviceAccountSecretRef:
          key: sa
          name: gcp-sa-secret
      selector:
        dnsZones:
        - 'test.example.com' # This should be the DNS name of the zone
```

每个单独的选择器块可以包含多个选择器类型，例如:

```yaml
solvers:
  - dns01:
    cloudflare:
      email: user@example.com
      apiKeySecretRef:
        name: cloudflare-apikey-secret
        key: apikey
    selector:
      matchLabels:
        "email": "user@example.com"
        "solver": "cloudflare"
      dnsZones:
        - "test.example.com"
        - "example.dev"
```

在这种情况下，Cloudflare 的`DNS01`求解器仅用于解决`Certificate`具有来自`matchLabels`的标签且 DNS 名称与来自`dnsZones`的区域匹配的 DNS 名称的挑战。

## 替代证书链

<a id="alternative-certificate-chain" className="hidden-link"></a>

从 ACME 服务器获取证书时，可以选择替代证书链。
这允许发行者在过渡期间优雅地将用户转到新的根证书上;最著名的例子是 Let's Encrypt[“ISRG 根”转换](https://community.letsencrypt.org/t/transition-to-isrgs-root-delayed-until-jan-11-2021/125516).

这个功能不是 Let’s Encrypt 独有的;如果您的 ACME 服务器支持多个 ca 的签名，您可以在证书的颁发者部分中使用`preferredChain`和您想要的链的 Common Name 的值。如果通用名与差值链匹配，服务器可以选择使用并返回该新链。

如果`preferredChain`与证书不匹配，服务器将返回它认为是默认证书的任何证书。

作为一个例子，下面是一个用户如何在(现在已经完成)之前请求一个替代链。
"ISRG Root"的转换，但请注意，因为这个改变已经发生了，所以没有必要再让我们加密了:

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    preferredChain: "ISRG Root X1"
```
