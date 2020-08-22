---
title: "CA 注射器(CA Injector)"
linkTitle: "CA注射器"
weight: 600
type: "docs"
---

该证书管理器 CA 注射器控制装置负责注入 CA 束入[webhook's](../webhook/) `ValidatingWebhookConfiguration` 和 `MutatingWebhookConfiguration` 资源 为了让 Kubernetes API 服务器 '相信' webhook API 服务器.

该组件使用 `cert-manager.io/inject-apiserver-ca: "true"` 和 `cert-manager.io/inject-ca-from: <NAMESPACE>/<CERTIFICATE>`
注释 在 `ValidatingWebhookConfiguration` 和 `MutatingWebhookConfiguration` resources 配置的.

它在整个 CA 副本 在定义 `cert-manager-webhook-ca` `Secret` 到了 `clientConfig.caBundle` 字段在两个 `ValidatingWebhookConfiguration` 和 `MutatingWebhookConfiguration` 资源 为了使 API 服务器信任他们各自的端点.

CA 喷射器可以作为独立的 pod 沿着侧的主证书管理器控制器和网络挂接部件.
