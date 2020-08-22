---
title: "备份和恢复资源"
linkTitle: ""
weight: 10
type: "docs"
description: >
  如果需要卸载证书管理器, 或者您的安装转移到新的集群, 你可以备份所有 cert-manager 的配置为了以后重新安装.
---

## 备份

为了您的 cert-manager 配置资源的所有备份, 运行:

```bash
$ kubectl get -o yaml \
     --all-namespaces \
     issuer,clusterissuer,certificates,certificaterequests > cert-manager-backup.yaml
```

如果你将数据传输到新的集群, 您可能还需要跨过额外 Secret 资源复制 由你的配置参考发行人, 如:

### CA Issuers

- 通过引用 `issuer.spec.ca.secretName` 根 CA Secret

### Vault Issuers

- 令牌认证 Secret 通过引用 `issuer.spec.vault.auth.tokenSecretRef`
- 该 AppRole 配置 Secret 通过引用 `issuer.spec.vault.auth.appRole.secretRef`

### ACME Issuers

- ACME 帐户私钥 Secret 通过引用 `issuer.acme.privateKeySecretRef`
- 任何 Secrets 通过引用 DNS 提供商 `issuer.acme.dns01.providers` 和 `issuer.acme.solvers.dns01` 字段下的配置 .

## 恢复资源

为了恢复配置, 你可以简单地 `kubectl apply` 文件 上面创建安装 cert-manager 后.

```bash
$ kubectl apply -f cert-manager-backup.yaml
```

如果你已经从一个旧的群集迁移, 你将也需要确保运行一个类似 `kubectl apply` 命令恢复您的 Secret 资源.
