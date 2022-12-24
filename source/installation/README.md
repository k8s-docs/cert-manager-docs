# 安装

了解可以安装 cert-manager 的各种方式，以及如何进行选择。

## 默认静态安装

> 不需要对 cert-manager 安装参数进行任何调整。

默认静态配置的安装方式如下:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.10.1/cert-manager.yaml
```

📖 阅读更多关于[使用 kubectl 应用程序和静态清单安装 cert-manager](./kubectl.md).

## 开始

> 您很快就会想要学习如何使用 cert-manager 以及它的用途。

📖 **kubectl apply**: 对于新用户，我们建议[使用 kubectl 应用程序和静态清单安装 cert-manager](./kubectl.md).

📖 **helm**: 您可以[使用 helm 来安装 cert-manager](./helm.md)，这也允许您在必要时自定义安装。

📖 **OperatorHub**: 如果您有一个 OpenShift 集群，请考虑[通过 OperatorHub 安装 cert-manager](./operator-lifecycle-manager.md)，您可以从 OpenShift web 控制台执行此操作。

🚧 **cmctl**: 尝试[experimental `cmctl x install` 命令](../reference/cmctl.md#install)来快速安装 cert-manager。

## 持续部署

> 您知道如何配置您的证书管理器设置，并希望将其自动化。

📖 **helm**: 您可以将[cert-manager Helm 图表](./helm.md)直接用于 Flux、ArgoCD 和 Anthos 等系统。

📖 **helm template**: 您可以使用`helm template`生成自定义的 cert-manager 安装清单。
参见[使用 helm 模板输出 YAML](./helm.md#output-yaml)了解更多细节。
这个模板化的证书管理器清单可以通过管道连接到您首选的部署工具中。
