# kubectl apply

学习如何使用 kubectl 和静态清单安装 cert-manager。

## 先决条件

- [安装 `kubectl` 版本 `>= v1.19.0`](https://kubernetes.io/docs/tasks/tools/). (否则，你将在更新 CRDs 时遇到问题 - 参见[v0.16 升级说明](./upgrading/upgrading-0.15-0.16.md#issue-with-older-versions-of-kubectl))
- 安装[受支持的 Kubernetes 或 OpenShift 版本](./supported-releases.md).
- 如果您正在云平台上使用 Kubernetes，请阅读[与 Kubernetes 平台提供商的兼容性](./compatibility.md)。

## 步骤

所有源( [`CustomResourceDefinitions`](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#customresourcedefinitions)和 cert-manager, caainjector 和 webhook 组件)都包含在单个 YAML 清单文件中:

安装所有 cert-manager 组件:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.10.1/cert-manager.yaml
```

默认情况下，cert-manager 将安装在`cert-manager`命名空间中。
可以在不同的名称空间中运行 cert-manager，不过需要对部署清单进行修改。

一旦部署了 cert-manager，就可以[验证安装](./verify.md).

## 谷歌 Kubernetes 引擎权限错误

当运行在 GKE(谷歌 Kubernetes 引擎)上时，您可能会在创建一些所需的源时遇到'permission denied'错误。
这是 GKE 处理 RBAC 和 IAM 权限的细微差别，因此，在运行`kubectl apply`之前，您可能需要将自己的权限提升到"cluster-admin"的权限。

如果你已经运行了`kubectl apply`，你应该在提升你的权限后再次运行它:

```bash
kubectl create clusterrolebinding cluster-admin-binding \
    --clusterrole=cluster-admin \
    --user=$(gcloud config get-value core/account)
```

## 卸载

!!! warning

    要卸载cert-manager，您应该始终使用与安装相同的过程，但是相反。
    无论从静态清单还是Helm安装cert-manager，偏离以下过程都可能导致问题和潜在的破坏状态。
    请确保在卸载时遵循以下步骤，以防止这种情况发生。

在继续之前，请确保用户创建的不需要的 cert-manager 源已经删除。
您可以使用以下命令检查任何现有的源:

```bash
kubectl get Issuers,ClusterIssuers,Certificates,CertificateRequests,Orders,Challenges --all-namespaces
```

建议在卸载 cert-manager 之前删除所有这些源。
如果计划稍后重新安装，并且不想失去一些自定义源，您可以保留它们。
然而，这可能会导致终结器出现问题。
一些源，比如`Challenges`，应该被删除，以避免[陷入挂起状态](#namespace-stuck-in-terminating-state).

一旦删除了不需要的源，就可以使用由安装方式决定的过程卸载 cert-manager 了。

!!! warning

    卸载证书管理器或简单地删除`Certificate`源可以导致TLS的`Secret`被删除，如果他们有`metadata.ownerReferences`设置的证书管理器。
    您可以使用`--enable-certificate-owner-ref` 控制器标志来控制是否将所有者引用添加到`Secret`中。
    默认情况下，该标志被设置为false，这意味着没有添加任何所有者引用。
    但是，在cert-manager v1.8及更老版本中，将标志的值从true更改为false _并不会_ 删除现有的所有者引用。
    在cert-manager v1.8中修复了此行为。
    请检查所有者引用，以确认它们确实被删除了。

### 使用常规清单卸载

从具有常规清单的安装中卸载是使用`kubectl`的`delete`命令*反向*运行安装过程的情况。

使用当前运行版本`vX.Y.Z`的链接删除安装清单，如下所示:

!!! Warning

    此命令还将删除已安装的cert-manager CRDs。
    所有证书管理器源(例如:`certificates.cert-manager.io`源)将被Kubernetes的垃圾收集器删除。
    如果删除`CustomResourceDefinition`，则不能保留任何自定义源。
    如果你想保留源，你应该单独管理`CustomResourceDefinition`。

```bash
kubectl delete -f https://github.com/cert-manager/cert-manager/releases/download/vX.Y.Z/cert-manager.yaml
```

### 命名空间处于终止状态

如果命名空间被标记为删除，而没有首先删除 cert-manager 安装，则命名空间可能会处于终止状态。
这通常是由于[`APIService`](https://kubernetes.io/docs/tasks/access-kubernetes-api/setup-extension-api-server)源仍然存在，但 webhook 不再运行，因此不再可达。
要解决这个问题，请确保正确运行了上述命令，如果仍然遇到问题，请运行:

```bash
kubectl delete apiservice v1beta1.webhook.cert-manager.io
```

#### 删除挂起的 Challenge

当终结器无法完成，而 Kubernetes 正在等待 cert-manager 控制器完成时，Challenge 可能会陷入悬而未决的状态。
当控制器不再运行以删除标志，并且源被定义为需要等待时，就会发生这种情况。
您可以通过手动执行控制器的操作来修复此问题。

首先，删除现有的 cert-manager webhook 配置，如果有的话:

```bash
kubectl delete mutatingwebhookconfigurations cert-manager-webhook
```

然后通过编辑 Challenge 源将`.metadata.finalizers`字段更改为空列表:

```bash
kubectl edit challenge <the-challenge>
```
