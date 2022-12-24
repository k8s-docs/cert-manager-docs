---
status: deprecated
---

# 可供选择的安装方法

## kubeprod(deprecated)

[Bitnami Kubernetes 生产运行时](https://github.com/bitnami/kube-prod-runtime) (`BKPR`, `kubeprod`) 是您需要部署在 Kubernetes 集群之上的服务的集合，以启用日志记录，监控，证书管理，通过公共 DNS 服务器自动发现 Kubernetes 源和其他公共基础设施需求。

它依赖于`cert-manager`进行证书管理，并且它是[定期测试的](https://github.com/bitnami/kube-prod-runtime/blob/master/Jenkinsfile)，因此已知组件在 GKE、AKS 和 EKS 集群中协同工作。
对于其入口堆栈，它在配置的 DNS 区域中创建一个 DNS 条目，并从 Let's Encrypt 登台服务器请求 TLS 证书。

可以使用`kubeprod install`命令部署 BKPR，该命令将部署`cert-manager`作为其一部分。
详细信息请参见[BKPR 安装指南](https://github.com/bitnami/kube-prod-runtime/blob/master/docs/install.md)。
