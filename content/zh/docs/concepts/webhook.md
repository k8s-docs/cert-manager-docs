---
title: "Webhook"
linkTitle: "网络挂接"
weight: 400
type: "docs"
---

`cert-manager` 利用使用`Webhook`服务器扩展 Kubernetes API 服务器在`CERT-manager`资源上提供[动态准入](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)控制.
这意味着，从大部分相同的行为的`CERT-manager`好处该有核心 Kubernetes 资源.
该`webhook`有三个主要功能:

- [`ValidatingAdmissionWebhook`](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#validatingadmissionwebhook):
  确保当创建或更新`CERT-manager`资源, 它们符合 API 规则.
  这种验证是更深入的比例如确保资源符合 OpenAPI 的架构, 而是包含逻辑，诸如不允许指定每个`Issuer`资源多于一个`Issuer`类型.
  该验证入场始终调用，将与成功或失败的响应响应.
- [`MutatingAdmissionWebhook`](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook):
  在创建和更新操作更改的内容资源，例如设置默认值.
- [`CustomResourceConversionWebhook`](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definition-versioning/#webhook-conversion):
  该`webhook`还负责实施过的版本转换 在 `cert-manager` `CustomResources` (`cert-manager.io`)里面.
  这意味着多个 API 版本可以支持, 如 `v1alpha2`, `v1alpha3`
  同时，使其能够依靠我们的配置模式的特定版本.

在`webhook`组件被部署为另一个`pod`该运行沿着主`CERT-manager`控制器和 CA 喷射器部件.

为了使 API 服务器与`webhook`部件通信, 网络挂接需要 TLS 证书的 API 服务器被配置为信任.
这是由[`cainjector`](../ca-injector/)建立并通过以下两个 Secrets 实施:

- `secret/cert-manager-webhook-ca`: 自签名的根 CA 证书这是用来签署证书的`webhook` `pod`.
- `secret/cert-manager-webhook-tls`: 由根 CA 上述发出的 TLS 证书, 由`webhook`服务.

如果周围出现错误的`webhook`但`webhook`运行 那么`webhook`是最有可能不是从 API 服务器可达.
在这种情况下, 确保 API 服务器可以通过以下的[GKE 专用群集说明](../../installation/compatibility/#gke)用`webhook`沟通 .
