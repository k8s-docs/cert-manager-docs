---
title: Tutorials
description: "cert-manager tutorials: Overview"
---

# 手册

一步一步的教程是开始使用 cert-manager 的好方法，我们提供了一些供您学习的教程。看看吧!

- [备份与恢复源](./backup.md): 备份集群中的 cert-manager 源，然后恢复它们。
- [Pomerium Ingress](./acme/pomerium-ingress.md): 使用证书管理器的 Pomerium 入口控制器教程。
- [使用 NGINX-Ingress 和 cert-manager 保护 ingress](./acme/nginx-ingress.md): 教程，用于将 NGINX 部署到集群中，并使用 Let's Encrypt 提供的证书保护传入连接。
- [使用 DNS 验证颁发 ACME 证书](./acme/dns-validation.md): 关于如何使用 DNS01 挑战解决 DNS 所有权验证的教程。
- [使用 HTTP 验证颁发 ACME 证书](./acme/http-validation.md): 关于如何使用 HTTP01 挑战解决 DNS 所有权验证的教程。
- [从 kube-lego 迁移过来](./acme/migrating-from-kube-lego.md): 关于如何从现在已弃用的 kube-lego 项目迁移的教程。
- [使用 Venafi 保护 EKS 集群](./venafi/venafi.md): 使用 Venafi 颁发的证书创建 EKS 集群和保护 NGINX 部署的教程。
- [使用 cert-manager 保护 Istio 服务网格](./istio-csr/istio-csr.md): 使用证书管理器颁发者保护 Istio 服务网格的教程。
- [跨命名空间同步秘密](./syncing-secrets-across-namespaces.md): 学习如何使用扩展(如:reflector, kubed 和 Kubernetes -replicator)跨命名空间同步 Kubernetes 秘密源。
- [使用 ZeroSSL 获取 SSL 证书](./zerossl/zerossl.md): 教程描述 ZeroSSL 作为外部 ACME 服务器的用法。

### 外部教程

- 一篇关于在 EKS 中使用 cert-manager 进行端到端加密的很棒的 AWS 博客文章。参见[在 Amazon EKS 上设置端到端 TLS 加密](https://aws.amazon.com/blogs/containers/setting-up-end-to-end-tls-encryption-on-amazon-eks-with-the-new-aws-load-balancer-controller/)
- GKE 集群上完整的证书管理器安装演示。参见[如何:Kubernetes 应用程序部署的自动 SSL 证书管理](https://medium.com/contino-engineering/how-to-automatic-ssl-certificate-management-for-your-kubernetes-application-deployment-94b64dfc9114)
- 使用工作负载标识在 GKE 集群上安装 cert-manager。参见[Kubernetes, ingress-nginx, cert-manager & external-dns](https://blog.atomist.com/kubernetes-ingress-nginx-cert-manager-external-dns/)
- 一个用于初学者的视频教程，展示了 cert-manager 的实际操作。参见[使用 cert-manager 为 Kubernetes 提供免费 SSL](https://www.youtube.com/watch?v=hoLUigg4V18)
