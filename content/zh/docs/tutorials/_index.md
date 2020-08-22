---
title: "教程"
linkTitle: ""
weight: 50
type: "docs"
---

强烈建议用户当使用证书管理器时熟悉的在文档中资源概念和配置.
然而, 一步一步的教程是对部署这些资源非常有用的得到一个最终目标.
下面的教程，你可能会发现有用的列表:

- [备份和恢复资源](./backup/): 在集群中，然后恢复这些备份的证书管理器的资源.
- [确保与入节点 NGINX，入口和证书管理器](./acme/ingress/): 教程 NGINX 部署到集群，并确保与 ACME 证书的连接.
- [发布使用 DNS 验证的 ACME 证书](./acme/dns-validation/): 教程如何使用 DNS01 挑战解析 DNS 所有权验证.
- [发行使用 ACME 证书 HTTP 验证](./acme/http-validation/): 教程如何使用 HTTP01 挑战解析 DNS 所有权验证.
- [从 KUBE-LEGO 迁移](./acme/migrating-from-kube-lego/): 教程如何从现在反对的 KUBE-LEGO 项目迁移.
- [保护的 EKS 集群与 Venafi](./venafi/venafi/): 教程创建 EKS 集群和一个 Venafi 确保 Nginx 上部署颁发的证书.

用户如何对博客和示例:

- 对 GKE 集群的完整证书管理器安装演示. 参见[操作方法：自动 SSL 证书管理进行 Kubernetes 应用程序部署](https://medium.com/contino-engineering/how-to-automatic-ssl-certificate-management-for-your-kubernetes-application-deployment-94b64dfc9114)
