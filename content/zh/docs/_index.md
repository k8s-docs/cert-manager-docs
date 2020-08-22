---
title: "欢迎来到 cert-manager"
linkTitle: "文档"
weight: 10
type: "docs"
---

cert-manager 是本地 [Kubernetes](https://kubernetes.io) 证书管理器.
它可以帮助从各种来源签发证书, 如[Let's Encrypt](https://letsencrypt.org), [HashiCorp Vault](https://www.vaultproject.io), [Venafi](https://www.venafi.com/), 一个简单的签名密钥对, 或自签名.

这将确保证书是有效的和最新的, 并试图在到期日之前配置的时间续订证书.

它是松散基于[kube-lego](https://github.com/jetstack/kube-lego)的工作并借鉴了其他类似项目的一些智慧如[kube-cert-manager](https://github.com/PalmStoneGames/kube-cert-manager).

![高级别概览图说明证书管理器架构](/images/high-level-overview.svg)

这是该项目的完整的技术文档, 并应与项目寻求帮助时，可以用作参考源.
