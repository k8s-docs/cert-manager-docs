---
title: "故障排除"
linkTitle: "故障排除"
weight: 60
type: "docs"
---

当故障排除证书管理器你最好的朋友是 `kubectl describe`, 这会给你以及对资源的信息最近发生的事件.
它不建议使用日志，这些都是相当冗长的，只应在如果以下步骤不提供帮助来看待.

CERT-经理由住你的 Kubernetes 集群内的多个自定义资源, 这些资源连接在一起，并且通常由彼此产生.
当这样的事件发生将被反映在 Kubernetes 事件, 你可以使用`kubectl get event`看到这些每个命名空间, 或当看着一个资源时在其输出 `kubectl describe` .

## 故障排除失败的证书请求

但是也有一些参与请求证书几个资源.

```

  (  +---------+  )
  (  | Ingress |  ) Optional                                              ACME Only!
  (  +---------+  )
         |                                                     |
         |   +-------------+      +--------------------+       |  +-------+       +-----------+
         |-> | Certificate |----> | CertificateRequest | ----> |  | Order | ----> | Challenge |
             +-------------+      +--------------------+       |  +-------+       +-----------+
                                                               |
```

该`cert-manager`在`Certificate`资源流中的所有启动, 如果你有[正确注释](../../usage/ingress/)集你可以创建这个自己或者您的入口资源将对这个给你.

### 1. 检查证书资源

首先我们要检查，我们是否在我们的命名空间中创建一个`Certificate`资源.
我们可以运用 `kubectl get certificate`得到这些 .

```console
$ kubectl get certificate
NAME                READY   AGE
example-com-tls     False   1h
```

如果两者都不存在，你打算使用[ingress-shim](../../usage/ingress/):
检查入口注解更多关于这在[入口排查引导](../../usage/ingress/#troubleshooting).
如果你没有使用 ingress-shim: 检查命令的输出用于创建证书.

如果你看到一个与准备状态 `False` 你可以运用 `kubectl describe certificate` 得到更多的信息 , 如果状态 `True` 这意味着，证书管理器已成功颁发证书.

```console
$ kubectl describe certificate <certificate-name>
[...]
Status:
  Conditions:
    Last Transition Time:        2020-05-15T21:45:22Z
    Message:                     Issuing certificate as Secret does not exist
    Reason:                      DoesNotExist
    Status:                      False
    Type:                        Ready
  Next Private Key Secret Name:  example-tls-wtlww
Events:
  Type    Reason     Age   From          Message
  ----    ------     ----  ----          -------
  Normal  Issuing    105s  cert-manager  Issuing certificate as Secret does not exist
  Normal  Generated  105s  cert-manager  Stored new private key in temporary Secret resource "example-tls-wtlww"
  Normal  Requested  104s  cert-manager  Created new CertificateRequest resource "example-tls-bw5t9"
```

在这里你会在`Status`下找到当前证书状态的详细信息以及详细信息和在`Events`下关于这是怎么回事.
双方将帮助您确定证书的当前状态.
最后的状态是 `Created new CertificateRequest resource`, 这是值得在该状态`CertificateRequest`考虑看看是让上为什么没有得到我们发出`Certificate`更多信息.

### 2. 检查 `CertificateRequest`

该`CertificateRequest`资源代表证书的管理器中的 CSR 并传递到发行者此 CSR。
您可以在`Certificate`事件日志中找到了`CertificateRequest`的名字或者使用 `kubectl get certificaterequest`

为了获得更多的信息我们再次运行 `kubectl describe`:

```console
$ kubectl describe certificaterequest <CertificateRequest name>
API Version:  cert-manager.io/v1alpha3
Kind:         CertificateRequest
Spec:
  Csr: [...]
  Issuer Ref:
    Group:  cert-manager.io
    Kind:   ClusterIssuer
    Name:   letencrypt-production
Status:
  Conditions:
    Last Transition Time:  2020-05-15T21:45:36Z
    Message:               Waiting on certificate issuance from order example-tls-fqtfg-1165244518: "pending"
    Reason:                Pending
    Status:                False
    Type:                  Ready
Events:
  Type    Reason        Age    From          Message
  ----    ------        ----   ----          -------
  Normal  OrderCreated  8m20s  cert-manager  Created Order resource example-tls-fqtfg-1165244518
```

在这里，我们将看到有关发行器的配置，以及发行器回应任何问题.

### 3. 检查发行状态

如果在上面的步骤，你看到了发行器没有准备好错误 你可以为（集群）发行器的资源做同样的步骤:

```console
$ kubectl describe issuer <Issuer name>
$ kubectl describe clusterissuer <ClusterIssuer name>
```

这将使你得到任何错误信息 关于与您的发卡行账户或网络问题.

### 4. ACME 故障排除

ACME (例如 Let's Encrypt) 发行器有 2 个额外的资源 里面证书管理器: `Orders` 和 `Challenges`.
故障排除这些被描述在[故障发出 ACME 证书](../acme/).
