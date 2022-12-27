---
title: 故障排除
description: 了解如何使用cert-manager调试常见问题
---

# 故障排除

在本节中，您将学习故障排除技术，这些技术将帮助您在证书颁发或更新失败时查找根本原因。

本节还包括以下指南:

- [解决 ACME / Let's Encrypt 证书问题](./acme.md): 了解有关 ACME 发布程序如何工作以及如何诊断问题的更多信息。
- [解决 Webhook 的问题](./webhook.md): 学习如何使用 cert-manager webhook 诊断问题。

## 概述

当对 cert-manager 进行故障排除时，您最好的朋友是`kubectl describe`，这将为您提供有关源以及最近事件的信息。
不建议使用日志，因为这些日志相当冗长，只有在以下步骤不能提供帮助时才应该查看。

cert-manager 由多个位于 Kubernetes 集群中的自定义源组成，这些源链接在一起，通常是由彼此创建的。
当这样的事件发生时，它将反映在 Kubernetes 事件中，您可以使用`kubectl get event`看到这些每个命名空间，或者在查看单个源时，在`kubectl describe`的输出中看到这些。

## 解决证书请求失败的问题

在请求证书时涉及到几个源。

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

证书管理器流都始于一个`Certificate`源，你可以自己创建这个源，如果你有[正确的注释](../usage/ingress.md)设置，你的 Ingress 源会为你做这件事。

### 1. 检查 Certificate 源

首先，我们必须检查是否在命名空间中创建了`Certificate`源。
我们可以使用`kubectl get certificate`来获得这些。

```console
$ kubectl get certificate
NAME                READY   AGE
example-com-tls     False   1h
```

如果没有，并且您计划使用[ingress-shim](../usage/ingress.md):请检查[ingress 故障排除指南](../usage/ingress.md#troubleshooting)中的入 ingress 注释。
如果您没有使用 ingress-shim:请检查创建证书命令的输出。

如果你看到一个就绪状态为`False`，你可以使用`kubectl describe certificate`获得更多信息，如果状态为`True`，这意味着证书管理器已经成功颁发了证书。

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

在这里，您将在`Status`下找到有关当前证书状态的更多信息，以及在`Events`下关于发生了什么的详细信息。两者都将帮助您确定证书的当前状态。
最后一个状态是`Created new CertificateRequest resource`，值得看看`CertificateRequest`处于哪个状态，以获得关于为什么我们的`Certificate`没有得到颁发的更多信息。

### 2. 检查 `CertificateRequest`

`CertificateRequest`源表示证书管理器中的 CSR，并将此 CSR 传递给颁发者。
您可以在`Certificate` 事件日志中找到`CertificateRequest`的名称，或者使用`kubectl get certificaterequest`

为了获得更多信息，我们再次运行`kubectl describe`:

```console
$ kubectl describe certificaterequest <CertificateRequest name>
API Version:  cert-manager.io/v1
Kind:         CertificateRequest
Spec:
  Request: [...]
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

在这里，我们将看到有关发行者配置和发行者响应的任何问题。

### 3. 检查发行者状态

如果在上面的步骤中，你看到一个发行者没有准备好错误，你可以对(集群)发行者源执行相同的步骤:

```console
$ kubectl describe issuer <Issuer name>
$ kubectl describe clusterissuer <ClusterIssuer name>
```

这将允许您获得有关您的发行者的帐户或网络问题的任何错误消息。
ACME 颁发者的故障诊断详见[颁发 ACME 证书的故障诊断](./acme.md)。

### 4. ACME 故障排除

ACME(例如，Let's Encrypt)发行者在证书管理器中有 2 个额外的源:`Orders` 和 `Challenges`。
在[颁发 ACME 证书的疑难解答](./acme.md)中描述了如何解决这些问题。
