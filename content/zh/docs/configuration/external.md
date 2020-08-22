---
title: "外部发行器"
linkTitle: "外部"
weight: 50
type: "docs"
---

cert-manager 支持外接 `Issuer` 类型.
这些外部 `Issuer` 类型是发行人 默认通过 cert-manager 不属于支持, 或者是 'out of tree', 然而被处理完全相同的任何其他内部 `Issuer` 类型. 外部发行人类型通常由部署另一个 pod 安装在支持群集将观看 `CertificateRequest` 资源和尊重他们根据配置 `Issuer` 资源.
这些发行人类型存在`cert-manager.io` 组外 .

作为 `v0.11`, 需要证书的管理器进行任何更改，以支持外部发行商.

这些外部发行人类型推荐的安装过程和配置选项可以在外部发行项目的文档中找到.
由它们的作者保持已知的外部发行人项目的清单如下:

- [step-issuer](https://github.com/smallstep/step-issuer): 用于请求来自[Smallstep](https://smallstep.com) [证书颁发机构服务器](https://github.com/smallstep/certificates)证书.

要创建自己的外部发行类型, 请按照[开发文档](../../contributing/external-issuers/)指导.
