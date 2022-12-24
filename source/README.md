---
title: cert-manager
description: cert-manager documentation homepage
---

# cert-manager

**cert-manager 文档主页**

cert-manager 在 Kubernetes 集群中增加了证书和证书颁发者作为源类型，简化了证书的获取、更新和使用过程。

它可以从各种受支持的来源颁发证书，包括[Let's Encrypt](https://letsencrypt.org)， [HashiCorp Vault](https://www.vaultproject.io)和[Venafi](https://www.venafi.com/)以及私有 PKI。

它将确保证书是有效的和最新的，并尝试在过期前的配置时间更新证书。

它松散地基于[kube-lego](https://github.com/jetstack/kube-lego)的工作，并借鉴了其他类似项目的一些智慧，如[kube-cert-manager](https://github.com/PalmStoneGames/kube-cert-manager)。

![解释证书管理器体系结构的高级概述图](/images/high-level-overview.svg)

本网站提供了项目的全部技术文件，可作为参考;如果您觉得有什么遗漏的，请告诉我们，或者[通过 PR](https://github.com/cert-manager/website/pulls)补充。
