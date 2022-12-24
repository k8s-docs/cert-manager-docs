---
title: Reference
description: 参考资料包括TLS术语、API文档和关于证书管理器组件的命令行标志的信息。
---

# 参考

本节包含参考资料，包括 TLS 术语、API 文档以及有关 cert-manager 组件的命令行标志的信息。

- [TLS 术语](./tls-terminology.md):
  了解证书管理器文档中使用的 TLS 术语，如`publicly trusted`, `self-signed`, `root`, `intermediate` 和 `leaf` _证书_。

- [组件/ Docker 映像](../cli/README.md):
  了解证书管理器 Docker 镜像的命令行标志:`controller`, `webhook`, `cainjector`, `acmesolver`，它们运行在集群的容器中。

- [API 参考](./api-docs.md):
  了解证书管理器 API，包括自定义资源，如 Certificate, CertificateRequest, Issuer 和 ClusterIssuer。
