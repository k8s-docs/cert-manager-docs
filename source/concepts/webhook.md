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

The [`cainjector`](./ca-injector.md) creates `secret/cert-manager-webhook-ca`, a self-signed root CA certificate which is used to sign certificates for the webhook pod.

Then the webhook can be configured with either

1. paths to a TLS certificate and key signed by the webhook CA, or
2. a reference to the CA Secret for dynamic generation of the certificate and key on webhook startup

## 已知问题及解决方法

### GKE 私有集群的 Webhook 连接问题

If errors occur around the webhook but the webhook is running then the webhook
is most likely not reachable from the API server. In this case, ensure that the
API server can communicate with the webhook by following the [GKE private
cluster explanation](../installation/compatibility.md#gke).

### AWS EKS 上的 Webhook 连接问题

When using a custom CNI (such as Weave or Calico) on EKS, the webhook cannot be reached by cert-manager.
This happens because the control plane cannot be configured to run on a custom CNI on EKS,
so the CNIs differ between control plane and worker nodes.
The solution is to [run the webhook in the host network](../installation/compatibility.md#aws-eks) so it can be reached by cert-manager.

### 安装 cert-manager 后不久出现 Webhook 连接问题

When you first install cert-manager, it will take a few seconds before the cert-manager API is usable.
This is because the cert-manager API requires the cert-manager webhook server, which takes some time to start up.
Here's why:

- The webhook server performs a leader election at startup which may take a few seconds.
- The webhook server may take a few seconds to start up and to generate its self-signed CA and serving certificate and to publish those to a Secret.
- `cainjector` performs a leader election at start up which can take a few seconds.
- `cainjector`, once started, will take a few seconds to update the `caBundle` in all the webhook configurations.

For these reasons, after installing cert-manager and when performing post-installation cert-manager API operations,
you will need to check for temporary API configuration errors and retry.

You could also add a post-installation check which performs `kubectl --dry-run` operations on the cert-manager API.
Or you could add a post-installation check which automatically retries the [Installation Verification](../installation/verify.md) steps until they succeed.

### 其他 Webhook 问题

If you encounter any other problems with the webhook, please refer to the [webhook troubleshooting guide](../troubleshooting/webhook/).
