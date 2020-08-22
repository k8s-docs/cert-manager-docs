---
title: "Kubectl 插件"
linkTitle: ""
weight: 100
type: "docs"
description: >
  `kubectl cert-manager` 是 [kubectl plugin](https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/) 它可以帮助你集群内管理 cert-manager 资源.
---

## 安装

您正在使用的平台您需要 `kubectl-cert-manager.tar.gz` 文件 , 这些都可以在我们的[GitHub 的版本页面](https://github.com/jetstack/cert-manager/releases)上找到.
为了使用 kubectl 插件 你需要它的二进制可访问 下名字`kubectl-cert_manager` 在你的 `$PATH`.
运行以下命令来设置插件:

```console
$ curl -L -o kubectl-cert-manager.tar.gz https://github.com/jetstack/cert-manager/releases/download/v0.16.1/kubectl-cert_manager-linux-amd64.tar.gz
$ tar xzf kubectl-cert-manager.tar.gz
$ sudo mv kubectl-cert_manager /usr/local/bin
```

您可以运行 `kubectl cert-manager help` 测试插件设置正确:

```console
$ kubectl cert-manager help

kubectl cert-manageris a CLI tool manage and configure cert-manager resources for Kubernetes

Usage:
  kubectl cert-manager [command]

Available Commands:
  convert     Convert cert-manager config files between different API versions
  create      Create cert-manager resources
  help        Help about any command
  renew       Mark a Certificate for manual renewal
  status      Get details on current status of cert-manager resources
  version     Print the kubectl cert-manager version

Use "kubectl cert-manager [command] --help" for more information about a command.
```

## 命令

### Renew

> **注意**: 对于 cert-manager `v0.15` 此功能需要 `ExperimentalCertificateControllers` 功能门集.
> 从 cert-manager `v0.16` 向前, 实验证控制器是默认.

`kubectl cert-manager renew` 可让您手动触发特定证书续期.
这可以在一个时间内完成任何一个证书, 使用标签选择器 (`-l app=example`), 或与 `--all` 标签:

例如，您可以续订证书 `example-com-tls`:

```console
$ kubectl get certificate
NAME                       READY   SECRET               AGE
example-com-tls            True    example-com-tls      1d

$ kubectl cert-manager renew example-com-tls
Manually triggered issuance of Certificate default/example-com-tls

$ kubectl get certificaterequest
NAME                              READY   AGE
example-com-tls-tls-8rbv2         False    10s
```

您也可以在更新给定命名空间中的所有证书:

```console
$ kubectl cert-manager renew --namespace=app --all
```

该 renew 命令允许指定几个选项:

- `--all` renew all Certificates in the given Namespace, or all namespaces when combined with `--all-namespaces`
- `-A` or `--all-namespaces` mark Certificates across namespaces for renewal
- `-l` `--selector` allows set a label query to filter on
  as well as `kubectl` global flags like `--context` and `--namespace`.

### Convert

`kubectl cert-manager convert` 可以用来转换 cert-manager 清单文件 不同的 API 版本之间.
无论 YAML 和 JSON 格式被接受.
该命令采用文件名, 目录, 或 URL 输入, 并通过 --output-version 标签将其转换为最新版本的格式或一个指定的 .

默认的输出将被被 YAML 格式打印到 stdout.
人们可以使用 -o 选项改变输出目的地.

例如，这将在最新版本的 API 里输出 `cert.yaml` :

```console
kubectl cert-manager convert -f cert.yaml
```

### Create

`kubectl cert-manager create` 可用于手动创建 cert-manager 资源.
子命令可用于创建不同的资源:

#### CertificateRequest

要创建 cert-manager CertificateRequest, 用 `kubectl cert-manager create certificaterequest`.
该命令将在 CertificateRequest 的名称创建, 并创建一个基于 YAML 清单的新 CertificateRequest 资源 由`--from-certificate-file`标志指定证书的资源, 通过在本地生成私钥和创造一个“证书签名请求” 提交到 cert-manager Issuer.
私钥将被写入到本地文件, 其中默认为 `<name_of_cr>.key`, 或者它可以用`--output-key-file`标志指定.

如果你希望等待该 CertificateRequest 要签名和存储 X509 证书到文件, 您可以设置`--fetch-certificate`标志.
默认的等待发放证书时，超时为 5 分钟, 但可以用`--timeout`标志指定.
该文件的存储 X509 证书的默认名称是 `<name_of_cr>.crt`, 你可以使用 `--output-certificate-file` 标签另行指定.

注意，私有密钥和 X509 证书都被写入到文件, 而**不**是保存在里面 Kubernetes.

例如，这将创建一个 CertificateRequest 资源 名为 "my-cr" 基于 cert-manager 证书描述 `my-certificate.yaml` 而存储私钥和 X509 证书 `my-cr.key` 和 `my-cr.crt` 分别.

```console
kubectl cert-manager create certificaterequest my-cr --from-certificate-file my-certificate.yaml --fetch-certificate --timeout 20m
```
