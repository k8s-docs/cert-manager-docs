# 与 Kubernetes 平台提供商的兼容性

_cert-manager 安装:云提供商兼容性_

下面您将发现部署 cert-manager 时可能会受到的各种兼容性问题和怪癖的详细信息。
如果你认为我们错过了一些东西，请随时提出一个问题或拉请求的细节!

如果您正在使用 AWS Fargate，或者您已经专门配置了 cert-manager 来运行主机的网络，请注意 kubelet 默认监听端口 `10250`，该端口与 cert-manager webhook 的默认端口冲突。

因此，在设置 cert-manager 时，您需要更改 webhook 的端口。

对于使用 Helm 的安装，您可以在安装 cert-manager 时使用命令行标志或`values.yaml`文件中的条目设置`webhook.securePort`参数。

如果端口冲突，您可能会看到关于不受信任的证书的令人困惑的错误消息。
详见[#3237](https://github.com/cert-manager/cert-manager/issues/3237)。

## GKE

当谷歌为私有集群配置控制平面时，它们会自动在 Kubernetes 集群的网络和单独的 Google 管理项目之间配置 VPC 对等。

In order to restrict what Google are able to access within your cluster, the
firewall rules configured restrict access to your Kubernetes pods. This means
that the webhook won't work, and you'll see errors such as
`Internal error occurred: failed calling admission webhook ... the server is
currently unable to handle the request`.

In order to use the webhook component with a GKE private cluster, you must
configure an additional firewall rule to allow the GKE control plane access to
your webhook pod.

You can read more information on how to add firewall rules for the GKE control
plane nodes in the [GKE
docs](https://cloud.google.com/kubernetes-engine/docs/how-to/private-clusters#add_firewall_rules).

### GKE Autopilot

GKE 自动驾驶模式与 Kubernetes < 1.21 不支持 cert-manager，由于[限制突变许可 webhooks](https://github.com/cert-manager/cert-manager/issues/3717).

As of October 2021, only the "rapid" Autopilot release channel has rolled
out version 1.21 for Kubernetes masters. Installation via the helm chart
may end in an error message but cert-manager is reported to be working by
some users. Feedback and PRs are welcome.

**Problem**: GKE Autopilot does not allow modifications to the `kube-system`-namespace.

Historically we've used the `kube-system` namespace to prevent multiple installations of cert-manager in the same cluster.

Installing cert-manager in these environments with default configuration can cause issues with bootstrapping.
Some signals are:

- `cert-manager-cainjector` logging errors like:

```text
E0425 09:04:01.520150       1 leaderelection.go:334] error initially creating leader election record: leases.coordination.k8s.io is forbidden: User "system:serviceaccount:cert-manager:cert-manager-cainjector" cannot create resource "leases" in API group "coordination.k8s.io" in the namespace "kube-system": GKEAutopilot authz: the namespace "kube-system" is managed and the request's verb "create" is denied
```

- `cert-manager-startupapicheck` not completing and logging messages like:

```text
Not ready: the cert-manager webhook CA bundle is not injected yet
```

**Solution**: Configure cert-manager to use a different namespace for leader election, like this:

```console
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version ${CERT_MANAGER_VERSION} --set global.leaderElection.namespace=cert-manager
```

## AWS EKS

当在 EKS 上使用自定义 CNI(如 Weave 或 Calico)时，证书管理器无法访问网络钩子。
这是因为控制平面不能配置为在 EKS 上的自定义 CNI 上运行，因此控制平面和工作节点之间的 CNI 是不同的。

为了解决这个问题，webhook 可以在主机网络中运行，这样 cert-manager 就可以访问它，方法是在你的部署中将`webhook.hostNetwork`键设置为 true，或者，如果使用 Helm，在你的`values.yaml`文件中配置它。

注意，在主机网络上运行将需要更改 webhook 的端口;有关详细信息，请参阅页面顶部的警告。

### AWS Fargate

值得注意的是，使用 AWS Fargate 不允许太多的网络配置，并且会导致 webhook 的端口与运行在端口 10250 上的 kubelet 冲突，如[#3237](https://github.com/cert-manager/cert-manager/issues/3237)所示。

在 Fargate 上部署 cert-manager 时，必须更改 webhook 侦听的端口。有关详细信息，请参阅本页顶部的警告。

因为 Fargate 强迫你使用它的网络，你不能手动设置网络类型和选项，如`webhook.hostNetwork`在 helm 图表上将导致您的证书管理器部署以令人惊讶的方式失败。
