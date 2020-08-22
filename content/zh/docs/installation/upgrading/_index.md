---
title: "升级"
linkTitle: ""
weight: 20
type: "docs"
---

本节包含在升级证书的管理器信息.
它还包含文件，详细说明证书管理器版本之间的重大更改, 和对事物的信息看出来，当升级.

> Note: Before performing upgrades of cert-manager, it is advised to take a
> backup of all your cert-manager resources just in case an issue occurs whilst
> upgrading. You can read how to backup and restore cert-manager in the [backup
> and restore](../../tutorials/backup/) guide.

## 用`Helm`升级

If you installed cert-manager using Helm, you can easily upgrade using the Helm
CLI.

> Note: Before upgrading, please read the relevant instructions at the links
> below for your from and to version.

Once you have read the relevant upgrading notes and taken any appropriate
actions, you can begin the upgrade process like so - replacing `<release_name>`
with the name of your Helm release for cert-manager (usually this is
`cert-manager`) and replacing `<version>` with the version number you want to
install:

If you have installed the CRDs manually instead of with the `--set installCRDs=true`
option added to your Helm install command, you should upgrade your CRD resources
before upgrading the Helm chart:

```bash
# Kubernetes 1.15+
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.16.1/cert-manager.crds.yaml

# Kubernetes <1.15
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.16.1/cert-manager-legacy.crds.yaml
```

Add the Jetstack Helm repository if you haven't already.

```bash
$ helm repo add jetstack https://charts.jetstack.io
```

## 确保本地`Helm`图表库高速缓存是最新的

```bash
$ helm repo update
$ helm upgrade --version <version> <release_name> jetstack/cert-manager
```

This will upgrade you to the latest version of cert-manager, as listed in the
[Jetstack Helm chart repository](https://hub.helm.sh/charts/jetstack).

> Note: You can find out your release name using `helm list | grep cert-manager`.

## 使用静态体现升级

If you installed cert-manager using the static deployment manifests published
on each release, you can upgrade them in a similar way to how you first
installed them.

> Note: Before upgrading, please read the relevant instructions at the links
> below Note: for your from and to version.

Once you have read the relevant notes and taken any appropriate actions, you can
begin the upgrade process like so - replacing `<version>` with the version
number you want to install:

```bash
# Kubernetes 1.15+
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.16.1/cert-manager.yaml

# Kubernetes <1.15
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.16.1/cert-manager-legacy.yaml
```
