---
title: Securing Istio Service Mesh
description: "cert-manager usage: Istio and istio-csr"
---

# 保护 Istio 服务网格

cert-manager 可以使用项目[istio-csr](https://github.com/cert-manager/istio-csr)与[Istio](https://istio.io)集成。
istio-csr 将部署一个代理，负责接收 Istio 网格所有成员的证书签名请求，并通过证书管理器对其进行签名。

[istio-csr](https://github.com/cert-manager/istio-csr)将通过您选择的证书管理器颁发者签署所有控制平面和工作负载证书。

---

请按照[项目页面](../projects/istio-csr)上的说明安装和使用 istio-csr.
