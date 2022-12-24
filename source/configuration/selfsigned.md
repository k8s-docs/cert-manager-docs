# SelfSigned

⚠️ `SelfSigned`颁发者通常用于在本地引导 PKI，这对于高级用户来说是一个复杂的主题。
为了在生产中安全使用，运行 PKI 会引入围绕轮换、信任存储库分发和灾难恢复的复杂规划需求。

如果您不打算运行自己的 PKI，请使用不同的颁发者类型。

`SelfSigned`颁发者并不代表证书颁发机构本身，而是表示证书将使用给定的私钥“对自己进行签名”。
换句话说，证书的私钥将用于对证书本身进行签名。

这种`Issuer`类型对于为自定义 PKI(公共密钥基础设施)引导根证书或创建简单的临时证书以进行快速测试非常有用。

有一些重要的[注意事项](#caveats) - 包括安全问题 - 需要考虑`SelfSigned`发行方;
一般来说，您可能希望使用[`CA`](./ca.md)颁发者而不是`SelfSigned`颁发者。
也就是说，`SelfSigned`颁发者对于最初引导一个`CA`颁发者非常有用。

!!! Note

    引用自签名证书的`CertificateRequest`也必须包含`cert-manager.io/private-key-secret-name`注释，因为需要`CertificateRequest`对应的私钥来签署证书。
    这个注释是由`Certificate`控制器自动添加的。

## 部署

由于`SelfSigned`颁发者不依赖于任何其他源，因此它是最简单的配置。
只有`SelfSigned`节需要出现在 Issuer spec 中，不需要其他配置:

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: selfsigned-issuer
  namespace: sandbox
spec:
  selfSigned: {}
```

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-cluster-issuer
spec:
  selfSigned: {}
```

部署后，您应该能够立即看到颁发者已准备好进行签名:

```bash
$ kubectl get issuers  -n sandbox -o wide selfsigned-issuer
NAME                READY   STATUS                AGE
selfsigned-issuer   True                          2m

$ kubectl get clusterissuers -o wide selfsigned-cluster-issuer
NAME                        READY   STATUS   AGE
selfsigned-cluster-issuer   True             3m
```

### 引导`CA`颁发者

`SelfSigned` 颁发者的理想用例之一是为私有 PKI 引导自定义根证书，包括使用证书管理器[`CA`](./ca.md)颁发者。

下面的 YAML 将创建一个`SelfSigned`颁发者，颁发一个根证书，并使用该根作为`CA`颁发者:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: sandbox
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-selfsigned-ca
  namespace: sandbox
spec:
  isCA: true
  commonName: my-selfsigned-ca
  secretName: root-secret
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
    group: cert-manager.io
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: my-ca-issuer
  namespace: sandbox
spec:
  ca:
    secretName: root-secret
```

### CRL 分发点

您还可以选择将[CRL](https://en.wikipedia.org/wiki/Certificate_revocation_list)分发点指定为字符串数组，每个字符串都标识 CRL 的位置，其中可以检查已颁发证书的撤销状态:

```yaml
---
spec:
  selfSigned:
    crlDistributionPoints:
      - "http://example.com"
```

## 警告

### 信任

客户端使用`SelfSigned`证书时，如果事先没有证书，就无法信任它们。
当使用证书的服务器的客户端存在于不同的名称空间中时，这就很难管理。
这个限制可以通过使用[trust-manager](../projects/trust-manager.md)将`ca.crt`分发到其他命名空间来解决。
另一种选择是使用"TOFU"(首次使用时信任)，这在发生中间人攻击时具有安全隐患。

### 证书的有效性

自签名证书的一个副作用是它的 Subject DN 和 Issuer DN 是相同的。
X.509 [RFC 5280，章节 4.1.2.4](https://tools.ietf.org/html/rfc5280#section-4.1.2.4)要求:

> 颁发者字段必须包含一个非空的区别名称(DN)。

但是，自签名证书在默认情况下没有设置主题 DN。
除非手动设置证书的主题 DN，否则颁发者 DN 将为空，从技术上讲，证书将是无效的。

规范中这一特定领域的验证是不完整的，并且在 TLS 库之间是不同的，但是如果您使用的证书带有空颁发者 DN，那么将来库将完全在规范中改进其验证并破坏您的应用程序，这始终存在风险。

为了避免这种情况，请确保为`SelfSigned` certs 设置一个主题。
这可以通过在证书管理器`Certificate`对象上设置`spec.subject`来实现，该对象将由`SelfSigned`颁发者颁发。

从 1.3 版开始，如果 cert-manager 检测到一个证书是由一个`SelfSigned`颁发者创建的，该颁发者 DN 为空，它将发出一个类型为`BadConfig`的 Kubernetes[警告事件](https://github.com/cert-manager/cert-manager/blob/45befd86966c563663d18848943a1066d9681bf8/pkg/controller/certificaterequests/selfsigned/selfsigned.go#L140)。
