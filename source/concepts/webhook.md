---
title: All About the cert-manager Webhook
description: 了解cert-manager的webhook组件，该组件验证、转换和设置cert-manager自定义源的默认值
---

# 关于证书管理器 Webhook

cert-manager 使用自定义源定义扩展 Kubernetes API。
它安装一个 webhook，有三个主要功能:

- [Validation](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#validatingadmissionwebhook):
  确保在创建或更新证书管理器源时，它们符合 API 的规则。
  这种验证比确保源符合 OpenAPI 模式更深入，而是包含了一些逻辑，例如不允许为每个“发行者”源指定多个“发行者”类型。
  验证许可总是被调用，并将以成功或失败的响应进行响应。
- [Mutation / Defaulting](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook):
  在创建和更新操作期间更改源的内容，例如设置默认值。
- [Conversion](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/#webhook-conversion):
  webhook 还负责在 cert-manager `CustomResources` (`cert-manager.io`)中实现版本转换。
  这意味着可以同时支持多个 API 版本;从`v1alpha2`到`v1`。
  这使得依赖于配置模式的特定版本成为可能。

!!! Tip "ℹ️ 这就是所谓的动态准入控制。"

    在 Kubernetes 文档中阅读更多关于[动态准入控制](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)的信息。

## 概述

webhook 组件作为另一个 pod 部署，与主要的 cert-manager 控制器和 CA 注入器组件一起运行。

为了让 API 服务器与 webhook 组件通信，webhook 需要 apisserver 配置为信任的 TLS 证书。

[`cainjector`](./ca-injector.md)创建了`secret/cert-manager-webhook-ca`，这是一个自签名根 CA 证书，用于为 webhook pod 签署证书。

然后 webhook 可以配置为任意一个

1. 由 webhook CA 签名的 TLS 证书和密钥的路径，或者
2. 用于在 webhook 启动时动态生成证书和密钥的 CA Secret 引用

## 已知问题及解决方法

### GKE 私有集群的 Webhook 连接问题

如果 webhook 周围发生了错误，但该 webhook 正在运行，那么该 webhook 很可能无法从 API 服务器访问。
在这种情况下，请按照[GKE 私有集群解释](../installation/compatibility.md#gke)确保 API 服务器可以与 webhook 通信。

### AWS EKS 上的 Webhook 连接问题

当在 EKS 上使用自定义 CNI(如 Weave 或 Calico)时，证书管理器无法访问该 webhook。
这是因为控制平面不能配置为在 EKS 上的自定义 CNI 上运行，所以控制平面和工作节点之间的 CNI 不同。
解决方案是[在主机网络中运行 webhook](../installation/compatibility.md#aws-eks)，这样 cert-manager 就可以访问它。

### 安装 cert-manager 后不久出现 Webhook 连接问题

当您第一次安装 cert-manager 时，需要几秒钟才能使用 cert-manager API。
这是因为 cert-manager API 需要 cert-manager webhook 服务器，这需要一些时间来启动。
原因如下:

- webhook 服务器在启动时执行 leader 选举，这可能需要几秒钟。
- webhook 服务器可能需要几秒钟的时间来启动并生成其自签名 CA 和服务证书，并将其发布到 Secret。
- `cainjector`在启动时执行领导选举，这可能需要几秒钟。
- `cainjector`, 一旦启动，将需要几秒钟更新所有 webhook 配置中的`caBundle`。

由于这些原因，在安装 cert-manager 之后以及在执行安装后的 cert-manager API 操作时，需要检查临时 API 配置错误并重试。

您还可以添加一个安装后检查，在 cert-manager API 上执行 `kubectl --dry-run`操作。
或者您可以添加安装后检查，自动重试[安装验证](../installation/verify.md)步骤，直到它们成功为止。

### 其他 Webhook 问题

如果您在 webhook 上遇到任何其他问题，请参考[webhook 故障排除指南](../troubleshooting/webhook/)。
