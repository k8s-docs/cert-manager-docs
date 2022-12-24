---
title: 备份与恢复源
description: "cert-manager tutorials: Backing up your cert-manager installation"
---

# 备份与恢复源

如果需要卸载 cert-manager，或将安装转移到新的集群，可以备份所有 cert-manager 的配置，以便稍后重新安装。

## 备份 cert-manager 资源配置

以下命令将备份`cert-manager`资源的配置。
在升级`cert-manager`之前，这样做可能很有用。
由于此备份不包括包含 X.509 证书的`Secrets`，因此还原到尚未拥有这些`Secrets`对象的集群将导致重新颁发证书。

### 备份

备份所有 cert-manager 配置资源，执行:

```bash
kubectl get --all-namespaces -oyaml issuer,clusterissuer,cert > backup.yaml
```

如果你要将数据传输到一个新的集群，你可能还需要复制其他的`Secret`源，这些源是由你配置的发行者引用的，例如:

#### CA Issuers

- 根 CA `Secret`由`issuer.spec.ca.secretName`引用

#### Vault Issuers

- The token authentication `Secret` referenced by `issuer.spec.vault.auth.tokenSecretRef`
- The AppRole configuration `Secret` referenced by `issuer.spec.vault.auth.appRole.secretRef`

#### ACME Issuers

- The ACME account private key `Secret` referenced by `issuer.acme.privateKeySecretRef`
- Any `Secret`s referenced by DNS providers configured under the `issuer.acme.dns01.providers` and `issuer.acme.solvers.dns01` fields.

### 恢复

为了恢复你的配置，你可以`kubectl apply`上面在安装 cert-manager 后创建的文件，除了`uid` 和 `resourceVersion`字段不需要恢复:

```bash
kubectl apply -f <(awk '!/^ *(resourceVersion|uid): [^ ]+$/' backup.yaml)
```

## 全集群备份和恢复

本节涉及备份和恢复集群中的“所有”Kubernetes 资源(包括一些`cert-manager`资源)，用于 isaster 恢复、集群迁移等场景。

!!! note

    我们已经在简单的Kubernetes测试集群上使用有限的Kubernetes发行版测试了这个过程。
    为了避免数据丢失，在生产中依赖备份和恢复策略之前，请在您自己的集群上测试备份和恢复策略。
    如果您遇到任何错误，请打开GitHub issue或PR，以记录不同Kubernetes环境下此过程的变化。

### 避免不必要的证书补发

#### 恢复顺序

If `cert-manager` does not find a Kubernetes `Secret` with an X.509 certificate
for a `Certificate`, reissuance will be triggered. To avoid unnecessary
reissuance after a restore, ensure that `Secret`s are restored before
`Certificate`s. Similarly, `Secret`s should be restored before `Ingress`es if you
are using [`ingress-shim`](../usage/ingress.md).

#### 从备份中排除一些证书管理器资源

`cert-manager` has a number of custom resources that are designed to represent a
point-in-time operation. An example would be a `CertificateRequest` that
represents a one-time request for an X.509 certificate. The status of these
resources can depend on other ephemeral resources (such as a temporary `Secret`
holding a private key) so `cert-manager` might not be able to correctly recreate
the state of these resources at a later point.

In most cases backup and restore tools will not restore the statuses of custom resources,
so including such one-time resources in a backup can result in an unnecessary reissuance
after a restore as without the status fields `cert-manager` will not be able to tell that,
for example, an `Order` has already been fulfilled.
To avoid unnecessary reissuance, we recommend that `Order`s and `Challenge`s are excluded
from the backup. We also don't recommend backing up `CertificateRequest`s, see [Backing up CertificateRequests](#backing-up-certificaterequests)

### 恢复入口证书

A `Certificate` created for an `Ingress` via [`ingress-shim`](../usage/ingress.md) will have an [owner
reference](https://kubernetes.io/docs/concepts/workloads/controllers/garbage-collection/#owners-and-dependents)
pointing to the `Ingress` resource. `cert-manager` uses the owner reference to
verify that the `Certificate` 'belongs' to that `Ingress` and will not attempt to
create/correct it for an existing `Certificate`. After a full
cluster recreation, a restored owner reference would probably be incorrect
(`Ingress` UUID will have changed). The incorrect owner reference could lead
to a situation where updates to the `Ingress` (i.e a new DNS name) are not
applied to the `Certificate`.

To avoid this issue, in most cases `Certificate`s created via `ingress-shim` can be excluded from the backup. Given that the restore happens
in the correct order (`Secret` with the X.509 certificate restored before the `Ingress`) `cert-manager` will be able to create a new `Certificate`
for the `Ingress` and determine that the existing `Secret` is for that `Certificate`.

### Velero

We have briefly tested backup and restore with `velero` `v1.5.3` and `cert-manager` versions `v1.3.1` and `v1.3.0` as well as `velero` `v1.3.1` and `cert-manager` `v1.1.0`.

A few potential edge cases:

- Ensure that the backups include `cert-manager` CRDs.
  For example, we have seen that if `--exclude-namespaces` flag is passed to `velero backup create`, CRDs for which there are no actual resources to be included in the backup might also not be included in backup unless `--include-cluster-resources=true` flag is also passed to the backup command.

- Velero does not restore statuses of custom resources, so you should probably exclude `Order`s, `Challenge`s and `CertificateRequest`s from the backup, see [Excluding some cert-manager resources from backup](#excluding-some-cert-manager-resources-from-backup).

- Velero's [default restore order](https://github.com/vmware-tanzu/velero/blob/main/pkg/cmd/server/server.go#L470)(`Secrets` before `Ingress`es, Custom Resources restored last), should ensure that there is no unnecessary certificate reissuance due to the order of restore operation, see [Order of restore](#order-of-restore).

- When restoring the deployment of `cert-manager` itself, it may be necessary to restore `cert-manager`'s RBAC resources before the rest of the deployment.
  This is because `cert-manager`'s controller needs to be able to create `Certificate`'s for the `cert-manager`'s webhook before the webhook can become ready.
  In order to do this, the controller needs the right permissions.
  Since Velero by default restores pods before RBAC resources, the restore might get stuck waiting for the webhook pod to become ready.

- Velero does not restore owner references, so it may be necessary to exclude `Certificate`s created for `Ingress`es from the backup even when not re-creating the `Ingress` itself. See [Restoring Ingress Certificates](#restoring-ingress-certificates).

## 备份 CertificateRequests

We no longer recommend including `CertificateRequest` resources in a backup for most scenarios.
`CertificateRequest`s are designed to represent a one-time request for an X.509 certificate. Once the request has been fulfilled,
`CertificateRequest` can usually be safely deleted[^1].
In most cases (such as when a `CertificateRequest` has been created for a `Certificate`) a new `CertificateRequest` will be created when needed (i.e at a time of a renewal of a `Certificate`).
In `v1.3.0` , as part of our work towards [policy implementation](https://github.com/cert-manager/cert-manager/pull/3727) we introduced identity fields for `CertificateRequest` resources where, at a time of creation, `cert-mananager`'s webhook updates `CertificateRequest`'s spec with immutable identity fields, representing the identity of the creator of the `CertificateRequest`.
This introduces some extra complexity for backing up and restoring `CertificateRequest`s as the identity of the restorer might differ from that of the original creator and in most cases a restored `CertificateRequest` would likely end up with incorrect state.

[^1]:
    there is an edge case where certain changes to `Certificate` spec may not
    trigger re-issuance if there is no `CertificateRequest` for that
    `Certificate`. See [documentation on when do certificates get
    re-issued](../faq/README.md#when-do-certs-get-re-issued).
