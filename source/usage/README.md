---
title: Issuing Certificates
description: "cert-manager usage: Overview"
---

# 签发证书

一旦配置了[`Issuer`](../configuration/README.md)，就可以颁发第一个证书了!

通过证书管理器请求证书有几种用例和方法:

- [证书的源](./certificate.md): 请求已签名证书的最简单和最常见的方法。
- [保护入口源](./ingress.md): 用于保护集群中的入口源的方法。
- [保护 OpenFaaS 功能](https://docs.openfaas.com/reference/ssl/kubernetes-with-cert-manager/): 使用 cert-manager 保护您的 OpenFaaS 服务。
- [与花园的融合](https://docs.garden.io/guides/cert-manager-integration): Garden 是一个开发 Kubernetes 应用程序的开发工具，它对集成证书管理器提供一流的支持。
- [确保 Knative](https://knative.dev/docs/serving/using-auto-tls/): 使用受信任的 HTTPS 证书保护您的 Knative 服务。
- [在有 CSI 的 Pods 上启用 mTLS](./csi.md): 使用证书管理器 CSI 驱动程序提供共享 pod 生命周期的唯一密钥和证书。
- [保护 Istio 网关](https://istio.io/docs/tasks/traffic-management/ingress/ingress-certmgr/): 使用 cert-manager 在 Kubernetes 中保护您的 Istio 网关。
- [保护 Istio 服务网格](./istio.md): 使用 cert-manager [Istio](https://istio.io)集成，通过 cert-manager 管理的证书保护每个 pod 的 mTLS PKI。
- [证书管理器证书的策略](./approver-policy.md): 通过自定义源定义策略管理哪些证书管理器证书可以签名或拒绝。
