---
title: "Kubernetes"
linkTitle: ""
weight: 20
type: "docs"
---

cert-manager 您 Kubernetes 集群中运行 作为一系列部署资源.
它利用 [`CustomResourceDefinitions`](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources) 配置证书颁发机构和证书请求.

它是利用常规 YAML 舱单部署, 像 Kubernetes 上任何其他应用程序.

一旦 cert-manager 已部署, 你必须配置 `Issuer` 或 `ClusterIssuer` 资源 代表证书颁发机构.
配置不同`Issuer`类型的详细信息 可以在[各自配置指南](../../configuration/)中找到.

> 注意: 从 cert-manager `v0.14.0` 向前, Kubernetes 的最低支持版本是 `v1.11.0`.
> 用户仍然在运行 Kubernetes `v1.10` 以下安装 cert-manager 前应该升级到支持的版本.

> **警告**: 你不应该一个集群上安装证书管理器的多个实例.
> 这将导致不确定的行为你可以从供应商被禁止 如 Let's Encrypt.

## 使用普通清单安装

所有资源 (`CustomResourceDefinitions`, cert-manager, namespace, 和 webhook 组件) 都包含在一个单一的 YAML 清单文件:

> **注意**: 如果您使用的是`kubectl`版本 在 `v1.19.0-rc.1` 以下 你将有更新`CRDs`问题.
> 欲了解更多信息请参阅[v0.16 升级说明](../upgrading/upgrading-0.15-0.16/#issue-with-older-versions-of-kubectl)

安装 `CustomResourceDefinitions` 和 cert-manager 本身:

```bash
# Kubernetes 1.15+
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.16.1/cert-manager.yaml

# Kubernetes <1.15
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.16.1/cert-manager-legacy.yaml
```

> **注意**: 如果您使用的是 Kubernetes 版本 在 `v1.15` 下你将需要安装清单的旧版本.
> 这个版本没有 API 的版本转换并且只支持 `cert-manager.io/v1alpha2` API 资源.

> **注意**: 如果您正在运行 Kubernetes `v1.15.4` 或以下, 你将需要添加 `--validate=false` 标签到您 `kubectl apply` 命令 以上 否则你会收到验证错误 与该 `x-kubernetes-preserve-unknown-fields` 字段 在 cert-manager 的 `CustomResourceDefinition` 资源里.
> 这是一个良性的错误，由于道路`kubectl`进行资源验证发生。

> **注意**: 当在 GKE 上运行 (谷歌 Kubernetes 引擎), 当创建一些资源时你可能会遇到一个 'permission denied' 错误.
> 这是 GKE 处理 RBAC 和 IAM 权限方式的细微差别 , 正因为如此，你应该 'elevate' 你自己的特权 以一个的 'cluster-admin' **之前** 运行上述命令.
> 如果您已经运行了上面的命令, 你应该提升您的权限后，再次运行它们:

```bash
  kubectl create clusterrolebinding cluster-admin-binding \
    --clusterrole=cluster-admin \
    --user=$(gcloud config get-value core/account)
```

> **Note**: 默认, cert-manager 将被安装到 `cert-manager` 命名空间.
> 它可以在不同的命名空间运行 cert-manager , 虽然你将需要修改部署清单.

一旦部署 cert-manager, 您可以验证安装[这里](./#verifying-the-installation).

## 用`Helm`安装

作为替代以上引用的清单 YAML, 我们还提供一个官方`Helm`图表安装`cert-manager`.

> **注意**: cert-manager 不应该被嵌入作为子图表到其它`Helm`图表.
> cert-manager 管理非命名空间资源在集群中并应只安装一次.

### 先决条件

- Helm v2 或 v3 已安装

### 注意: Helm v2

在使用 Helm v2 部署之前 cert-manager , 你必须确保 [Tiller](https://github.com/helm/helm) 启动并在集群中运行.
Tiller 是服务器端组件至 Helm.

群集管理员可能已经为你设置和配置 Helm, 在这种情况下，你可以跳过这一步.

安装 Helm 上完整文档可以被找寻到在[安装 helm 文档](https://v2.helm.sh/docs/install/#installing-helm)里面.

如果群集具有 RBAC (Role Based Access Control) 启用 (default in GKE `v1.7`+), 当部署 Tiller 你将需要采取特殊照顾, 确保`Tiller`有权创建资源作为群集管理员.
使用 RBAC 部署 Helm 更多信息可以在[Helm RBAC 文档](https://github.com/helm/helm/blob/240e539cec44e2b746b3541529d41f4ba01e77df/docs/rbac.md#Example-Service-account-with-cluster-admin-role)找到.

### 步骤

为了安装 Helm 图表, 你必须遵循以下步骤:

创建`cert-manager`命名空间:

```bash
$ kubectl create namespace cert-manager
```

添加`Jetstack` `Helm`库:

> **警告**: 重要的是，这个仓库是用来安装 cert-manager.
> 该版本驻留在 helm 稳定的资源库里是 _已过时_ 而 _不_ 应使用.

```bash
$ helm repo add jetstack https://charts.jetstack.io
```

更新本地`Helm`图表库高速缓存:

```bash
$ helm repo update
```

cert-manager 需要大量 CRD 资源被安装到您的群集安装的一部分.

当安装`Helm`图表时这既可以运用 `kubectl`手动完成, 或使用 `installCRDs` 选项 .

> **注意**: 如果您使用的是 `helm` 基于 Kubernetes 版本 `v1.18` 以下 (Helm `v3.2`) `installCRDs`不会与工作 cert-manager `v0.16`.
> 欲了解更多信息请参阅[v0.16 升级注意事项](../upgrading/upgrading-0.15-0.16/#helm)

**选项 1: 安装 CRDs with `kubectl`**

运用 `kubectl`安装 `CustomResourceDefinition` 资源 :

```bash
# Kubernetes 1.15+
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.16.1/cert-manager.crds.yaml

# Kubernetes <1.15
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.16.1/cert-manager-legacy.crds.yaml
```

> **注意**: 如果您使用的是 Kubernetes 版本 `v1.15` 以下 你将需要安装顶层要求的旧版本.
> 这个版本没有 API 版本转变并且只支持 `cert-manager.io/v1alpha2` API 资源.

**选项 2: 安装 CRDs 作为`Helm`版本的一部分**

自动安装 和 管理顶层要求为你的`Helm`发行版的一部分, 您必须添加 `--set installCRDs=true` 标签到您的 Helm 安装命令.

取消注释在接下来的步骤中的相关行启用此.

---

要安装 cert-manager Helm 图表:

```bash
# Helm v3+
$ helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v0.16.1 \
  # --set installCRDs=true

# Helm v2
$ helm install \
  --name cert-manager \
  --namespace cert-manager \
  --version v0.16.1 \
  jetstack/cert-manager \
  # --set installCRDs=true
```

默认 cert-manager 配置有利于广大用户, 但可用选项的完整列表可以在[Helm 图自述](https://hub.helm.sh/charts/jetstack/cert-manager)找到.

## 验证安装

一旦你安装 cert-manager, 你可以验证它是否正确部署 通过检查 `cert-manager` 运行 pods 命名空间 :

```bash
$ kubectl get pods --namespace cert-manager

NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-5c6866597-zw7kh               1/1     Running   0          2m
cert-manager-cainjector-577f6d9fd7-tr77l   1/1     Running   0          2m
cert-manager-webhook-787858fcdb-nlzsq      1/1     Running   0          2m
```

您应该看到 `cert-manager`, `cert-manager-cainjector`, 和 `cert-manager-webhook` pod 在一个 `Running` 状态.
这可能需要一分钟左右为 TLS 资产 所需 webhook 以功能置备.
这可能会导致 webhook 需要一段时间更长的时间来启动 首次高于其他 pods.
如果遇到问题, 请检查[常见问题指南](../../faq/).

下面的步骤将确认 cert-manager 设置正确并且能够发出基本的证书类型.

创建 `Issuer` 测试 webhook 工作好.

```bash
$ cat <<EOF > test-resources.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager-test
---
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: test-selfsigned
  namespace: cert-manager-test
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: selfsigned-cert
  namespace: cert-manager-test
spec:
  dnsNames:
    - example.com
  secretName: selfsigned-cert-tls
  issuerRef:
    name: test-selfsigned
EOF
```

创建测试资源.

```bash
$ kubectl apply -f test-resources.yaml
```

检查新创建的证书的状态.
在 cert-manager 处理该证书请求之前您可能需要等待几秒钟.

```bash
$ kubectl describe certificate -n cert-manager-test

...
Spec:
  Common Name:  example.com
  Issuer Ref:
    Name:       test-selfsigned
  Secret Name:  selfsigned-cert-tls
Status:
  Conditions:
    Last Transition Time:  2019-01-29T17:34:30Z
    Message:               Certificate is up to date and has not expired
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2019-04-29T17:34:29Z
Events:
  Type    Reason      Age   From          Message
  ----    ------      ----  ----          -------
  Normal  CertIssued  4s    cert-manager  Certificate issued successfully
```

清理测试资源.

```bash
$ kubectl delete -f test-resources.yaml
```

如果所有上述步骤完成没有错误, 你已准备好出发!

如果遇到问题, 请检查[常见问题](../../faq/).

## 配置您的第一个发行器

在您开始颁发证书, 你必须在集群中至少配置一个`Issuer`或`ClusterIssuer`资源 .

你应该阅读[配置](../../configuration/) 指南学习如何配置 cert-manager 从支持后端的一个颁发证书 .

## 安装 kubectl 插件

cert-manager 也有 kubectl 插件它可以用来帮助你管理集群中 cert-manager 资源 .
此安装说明可以在[kubectl plugin](../../usage/kubectl-plugin/) 文档找到.

## 其它安装方法

### kubeprod

[Bitnami Kubernetes 生产运行时](https://github.com/bitnami/kube-prod-runtime) (`BKPR`, `kubeprod`) 是服务的集合策划 你需要在你的 Kubernetes 集群的顶部部署启用日志记录, 监控, 证书管理,Kubernetes 资源的自动发现 通过公共 DNS 服务器和其他常见的基础设施需求.

这取决于 `cert-manager` 证书管理, 它是 [定期测试](https://github.com/bitnami/kube-prod-runtime/blob/master/Jenkinsfile) 因此，组件是众所周知的协同工作 对 GKE 和 AKS 集群 (EKS 即将加入).
对于其进入堆栈它会在配置 DNS 区域的 DNS 条目 并要求从咱们的加密升级服务器 TLS 证书.

BKPR 可以使用`kubeprod install`命令进行部署, 这将部署 `cert-manager` 作为它的一部分.
在[BKPR 安装指南](https://github.com/bitnami/kube-prod-runtime/blob/master/docs/install.md)详细提供.

### 调试安装问题

如果你有安装的任何问题, 请参阅[常见问题](../../faq/).
