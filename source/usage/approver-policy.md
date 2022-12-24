---
title: Policy for cert-manager certificates
description: "cert-manager usage: approver-policy"
---

cert-manager [CertificateRequests](../concepts/certificaterequest.md)可以通过使用[approval API](../concepts/certificaterequest.md#approval)拒绝被签名。
[approver-policy](https://github.com/cert-manager/approver-policy)是一个证书管理器项目，允许您编写策略来自动管理此审批机制。

有关如何安装和使用 approver-policy 的更多信息，请阅读[项目页面](../projects/approver-policy.md)。
