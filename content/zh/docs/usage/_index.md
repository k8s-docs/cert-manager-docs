---
title: "用法"
linkTitle: ""
weight: 40
type: "docs"
---

## 使用证书管理器颁发证书

一旦 [`Issuers`](../configuration/) 已配置, 证书能够通过这些'Issuers`请求，并签署.
下面显示的是通过证书管理器申请证书的用例和方法的列表 :

- [证书资源](./certificate/): 请求签名证书的最简单和最常用的方法.
- [保护入口资源](./ingress/): 一种方法到集群中的保护入口资源.
- [保护 OpenFaaS 功能](https://docs.openfaas.com/reference/ssl/kubernetes-with-cert-manager/): 使用证书管理器保护您的 OpenFaaS 服务.
- [集成 Garden](https://docs.garden.io/guides/cert-manager-integration): Garden 对于开发 Kubernetes 应用程序的开发人员工具 这对于整合`CERT-manager`一流的支持.
- [保护 Knative](https://knative.dev/docs/serving/using-auto-tls/): 与信任的 HTTPS 证书保护您的 Knative 服务.
- [使用 CSI 启用 Pods MTLS ](./csi/): 使用证书管理器 CSI 驱动提供独特的密钥和证书 该共享 pods 生命周期 .
- [保护 Istio 网关](https://istio.io/docs/tasks/traffic-management/ingress/ingress-certmgr/): 使用证书管理器保护您的 Kubernetes 上 Istio 网关.
- [保护 Istio 服务网](./istio/): 使用 cert-manager [Istio](https://istio.io) 集成, 保护 MTLS PKI 每个 pod 通过证书管理器管理证书.
