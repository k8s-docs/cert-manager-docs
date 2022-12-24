---
title: CertificateRequest
description: "cert-manager core concepts: CertificateRequests"
---

# CertificateRequest

`CertificateRequest`是证书管理器中的命名空间源，用于向 [`Issuer`](./issuer.md)请求 X.509 证书。
该源包含一个 base64 编码的 PEM 编码的证书请求字符串，该字符串被发送到引用的颁发者。
成功的颁发将根据证书签名请求返回已签名的证书。
`CertificateRequests`通常由控制器或其他系统使用和管理，除非特别需要，否则不应由人类使用。

一个简单的 `CertificateRequest` 如下所示:

```yaml
apiVersion: cert-manager.io/v1
kind: CertificateRequest
metadata:
  name: my-ca-cr
spec:
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQzNqQ0NBY1lDQVFBd2daZ3hDekFKQmdOVkJBWVRBbHBhTVE4d0RRWURWUVFJREFaQmNHOXNiRzh4RFRBTApCZ05WQkFjTUJFMXZiMjR4RVRBUEJnTlZCQW9NQ0VwbGRITjBZV05yTVJVd0V3WURWUVFMREF4alpYSjBMVzFoCmJtRm5aWEl4RVRBUEJnTlZCQU1NQ0dwdmMyaDJZVzVzTVN3d0tnWUpLb1pJaHZjTkFRa0JGaDFxYjNOb2RXRXUKZG1GdWJHVmxkWGRsYmtCcVpYUnpkR0ZqYXk1cGJ6Q0NBU0l3RFFZSktvWklodmNOQVFFQkJRQURnZ0VQQURDQwpBUW9DZ2dFQkFLd01tTFhuQkNiRStZdTIvMlFtRGsxalRWQ3BvbHU3TlZmQlVFUWl1bDhFMHI2NFBLcDRZQ0c5Cmx2N2kwOHdFMEdJQUgydnJRQmxVd3p6ZW1SUWZ4YmQvYVNybzRHNUFBYTJsY2NMaFpqUlh2NEVMaER0aVg4N3IKaTQ0MWJ2Y01OM0ZPTlRuczJhRkJYcllLWGxpNG4rc0RzTEVuZmpWdXRiV01Zeis3M3ptaGZzclRJUjRzTXo3cQpmSzM2WFM4UkRjNW5oVVcyYU9BZ3lnbFZSOVVXRkxXNjNXYXVhcHg2QUpBR1RoZnJYdVVHZXlZUUVBSENxZmZmCjhyOEt3YTFYK1NwYm9YK1ppSVE0Nk5jQ043OFZnL2dQVHNLZmphZURoNWcyNlk1dEVidHd3MWdRbWlhK0MyRHIKWHpYNU13RzJGNHN0cG5kUnRQckZrU1VnMW1zd0xuc0NBd0VBQWFBQU1BMEdDU3FHU0liM0RRRUJDd1VBQTRJQgpBUUFXR0JuRnhaZ0gzd0N3TG5IQ0xjb0l5RHJrMUVvYkRjN3BJK1VVWEJIS2JBWk9IWEFhaGJ5RFFLL2RuTHN3CjJkZ0J3bmlJR3kxNElwQlNxaDBJUE03eHk5WjI4VW9oR3piN0FVakRJWHlNdmkvYTJyTVhjWjI1d1NVQmxGc28Kd005dE1QU2JwcEVvRERsa3NsOUIwT1BPdkFyQ0NKNnZGaU1UbS9wMUJIUWJSOExNQW53U0lUYVVNSFByRzJVMgpjTjEvRGNMWjZ2enEyeENjYVoxemh2bzBpY1VIUm9UWmV1ZEp6MkxmR0VHM1VOb2ppbXpBNUZHd0RhS3BySWp3ClVkd1JmZWZ1T29MT1dNVnFNbGRBcTlyT24wNHJaT3Jnak1HSE9tTWxleVdPS1AySllhaDNrVDdKU01zTHhYcFYKV0ExQjRsLzFFQkhWeGlKQi9Zby9JQWVsCi0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=
  isCA: false
  usages:
    - signing
    - digital signature
    - server auth
  # 90 days
  duration: 2160h
  issuerRef:
    name: ca-issuer
    # We can reference ClusterIssuers by changing the kind here.
    # The default value is Issuer (i.e. a locally namespaced Issuer)
    kind: Issuer
    group: cert-manager.io
```

这个`CertificateRequest`将使证书管理器尝试请求默认颁发者组`cert-manager.io`中的`Issuer` `ca-issuer`，并根据证书签名请求返回一个证书。
其他组可以在`issuerRef`中指定，它将目标发行者更改为您可能已安装的其他外部第三方发行者。

该源还公开了将证书声明为 CA 的选项、密钥用法和请求的有效期限。

`CertificateRequest`的`spec`中的所有字段，以及任何托管的证书管理器注释，都是不可变的，在创建后不能修改。

证书签名请求的成功颁发将导致对源的更新，使用已签名的证书、证书的 CA(如果可用)设置状态，并将`Ready`条件设置为`True`。

无论证书签名请求的颁发是否成功，都不会重试颁发。其他控制器负责管理`CertificateRequests`的逻辑和生命周期。

## 条件

`CertificateRequests`有一组强定义的条件，控制器或服务应该使用和依赖这些条件来决定下一步对源采取什么行动。

### Ready

每个就绪条件由`Ready`-一个布尔值和`Reason`-一个字符串组成。
值和含义的集合如下:

| Ready | Reason  | 条件的意义                                                                                                                            |
| ----- | ------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| False | Pending | `CertificateRequest`目前正在等待，等待一些其他操作发生。这可能是“颁发者”还不存在，或者“颁发者”正在颁发证书。                          |
| False | Failed  | 颁发证书失败 - 要么是返回的证书未能解码，要么是用于签名的引用颁发者的实例失败。它的控制器不会对`CertificateRequest`采取进一步的操作。 |
| True  | Issued  | 已由引用的“颁发者”成功颁发签名证书。                                                                                                  |

## UserInfo

`CertificateRequests`包含一组`UserInfo`字段作为规范的一部分，即:`username`, `groups`, `uid`, 和 `extra`。
这些值包含了创建`CertificateRequests`的用户。
如果`CertificateRequests`是由[`Certificate`](./certificate.md) 源创建的，则该用户将是证书管理器本身，或者直接创建`CertificateRequests`的用户。

!!! Warning

    这些字段由cert-manager管理，不应该被其他任何东西设置或修改。
    当`CertificateRequests`被创建时，这些字段将被覆盖，任何试图修改它们的请求将被拒绝。

### Approval 批准

证书请求可以被`Approved` or `Denied`.。
这些互斥条件阻止 CertificateRequest 被其托管签署人签署。

- 签署人 _不_ 应在没有已批准条件的托管证书请求上签字
- 签名者 _将_ 使用已批准的条件签署托管的 CertificateRequest
- 签名者 _永远_ 不会签署具有 Denied 条件的托管 CertificateRequest

这些条件是 _永久性的_，一旦设置就不能修改或更改。

```bash
NAMESPACE      NAME                    APPROVED   DENIED   READY   ISSUER       REQUESTOR                                         AGE
istio-system   service-mesh-ca-whh5b   True                True    mesh-ca      system:serviceaccount:istio-system:istiod         16s
istio-system   my-app-fj9sa                       True             mesh-ca      system:serviceaccount:my-app:my-app               4s
```

#### Behavior

Approved 条件和 Denied 条件是 CertificateRequest 上的两种不同的条件类型。
这些条件必须只具有 True 状态，并且是互斥的(即 CertificateRequest 不能同时具有 Approved 和 Denied 条件)。
此行为在证书管理器验证许可 webhook 中强制执行。

"approver"是负责设置批准/拒绝条件的实体。
由审批者管理哪些 CertificateRequests，这取决于审批者的实现。

“批准/拒绝”条件的“原因”字段应设置为“ _谁_ 设置条件”。
但是，谁可以被解释，对于审批者实现是有意义的。
例如，它可能包括审批策略控制器的 API 组，或者手动请求的客户端代理。

“批准/拒绝”条件的 Message 字段应该设置为设置条件的原因。
同样，为什么可以解释，但对审批者的实现是有意义的。
例如，批准此请求的源的名称，导致请求被拒绝的违规行为，或者手动批准该请求的团队。

#### Approver Controller

默认情况下，cert-manager 将运行一个内部审批控制器，它将自动批准所有引用任何命名空间中的任何内部颁发者类型的 CertificateRequests: `cert-manager.io/Issuer`, `cert-manager.io/ClusterIssuer`。

要禁用此控制器，请在 cert-manager-controller 中添加以下参数:`--controllers=*,-certificaterequests-approver`。
这可以通过添加 `helm` 来实现:

```bash
--set extraArgs={--controllers='*\,-certificaterequests-approver'}
```

或者，为了让内部审核者控制器批准引用外部颁发者的`CertificateRequests`，将以下 RBAC 添加到 cert-manager-controller 服务帐户。
请将给定的源名称替换为相关名称:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cert-manager-controller-approve:my-issuer-example-com # edit
rules:
  - apiGroups:
      - cert-manager.io
    resources:
      - signers
    verbs:
      - approve
    resourceNames:
      - issuers.my-issuer.example.com/* # edit
      - clusterissuers.my-issuer.example.com/* # edit
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cert-manager-controller-approve:my-issuer-example-com # edit
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cert-manager-controller-approve:my-issuer-example-com # edit
subjects:
  - kind: ServiceAccount
    name: cert-manager
    namespace: cert-manager
```

#### RBAC 语法

当用户或控制器试图批准或拒绝一个 CertificateRequest 时，证书管理器 webhook 将评估它是否有足够的权限这样做。
这些权限是基于请求本身的-特别是请求的 IssuerRef:

```yaml
apiGroups: ["cert-manager.io"]
resources: ["signers"]
verbs: ["approve"]
resourceNames:
  # namesapced signers
  - "<signer-resource-name>.<signer-group>/<signer-namespace>.<signer-name>"
  # cluster scoped signers
  - "<signer-resource-name>.<signer-group>/<signer-name>"
  # all signers of this resource name
  - "<signer-resource-name>.<signer-group>/*"
```

一个示例 ClusterRole 将授予权限来设置`CertificateRequests`的批准和拒绝条件，引用集群作用域的`myissuers`外部颁发者，在组`my-exampleio`中，名称为`myapp`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: my-example-io-my-issuer-myapp-approver
rules:
  - apiGroups: ["cert-manager.io"]
    resources: ["signers"]
    verbs: ["approve"]
    resourceNames: ["myissuers.my-example.io/myapp"]
```

如果审批者没有上面定义的足够的权限来设置批准或拒绝条件，请求将被验证许可的证书管理器拒绝。

- RBAC 权限必须在集群范围内授予
- 具有名称空间的签名者由具有名称空间的源使用 `<signer-resource-name>.<signer-group>/<signer-namespace>.<signer-name>` 语法表示
- 集群作用域的签名者使用 `<signer-resource-name>.<signer-group>/<signer-name>` 的语法表示
- 可以通过 `<signer-resource-name>.<signer-group>/*` 向审批者授予所有名称空间的审批权
- apiGroup 必须 _总_ 是 `cert-manager.io`
- 源必须 _总_ 是`signers`
- 动词必须 _总_ 是`approve`，它授予审批者设置“Approved”和“Denied”条件的权限

在`my-example.io`组中为所有命名空间中的所有`myissuer`签名者签名，以及为`clustermyissuers`签名的名称为 `myapp` 的示例:

```yaml
resourceNames: ["myissuers.my-example.io/*", "clustermyissuers.my-example.io/myapp"]
```

在命名空间`foo` 和 `bar`中使用`myapp`为`myissuer`签名:

```yaml
resourceNames: ["myissuers.my-example.io/foo.myapp", "myissuers.my-example.io/bar.myapp"]
```
