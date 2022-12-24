---
title: trust-manager
description: "Distributing Trust Bundles in Kubernetes"
---

# trust-manager

## 在 Kubernetes 中分发信任包

trust-manager 是用于在 Kubernetes 集群中分发信任包的操作符。
trust-manager 旨在补充[cert-manager](https://github.com/cert-manager/cert-manager)，使服务能够信任由发行者签署的 X.509 证书，以及证书管理器可能根本不知道的外部 CAs。

## 使用

trust 附带了一个单一集群范围的`Bundle`源。
一个`Bundle`表示一组应该在整个集群中分布和可用的数据。
对于可以分发的数据没有任何限制。

Bundle 从位于信任名称空间(信任控制器部署在其中)的许多“源”收集并追加信任数据，并将它们同步到每个名称空间中的`target`。

典型的 Bundle 如下所示:

```yaml
apiVersion: trust.cert-manager.io/v1alpha1
kind: Bundle
metadata:
  name: my-org.com
spec:
  sources:
    # A Secret in the trust namespace created via a cert-manager Certificate
    - secret:
        name: "my-db-tls"
        key: "ca.crt"
    # A ConfigMap in the trust namespace
    - configMap:
        name: "my-org.net"
        key: "root-certs.pem"
    # An In Line
    - inLine: |
        # my-org.com CA
        -----BEGIN CERTIFICATE-----
        MIIC5zCCAc+gAwIBAgIBADANBgkqhkiG9w0BAQsFADAVMRMwEQYDVQQDEwprdWJl
        ....
        0V3NCaQrXoh+3xrXgX/vMdijYLUSo/YPEWmo
        -----END CERTIFICATE-----
  target:
    # Data synced to the ConfigMap `my-org.com` at the key `root-certs.pem` in
    # every namespace that has the label "linkerd.io/inject=enabled".
    configMap:
      key: "root-certs.pem"
    namespaceSelector:
      matchLabels:
        linkerd.io/inject: "enabled"
```

Bundle 目前支持源类型`configMap`, `secret` 和 `inLine`,目标类型`configMap`。

#### 名称空间选择器

目标`namespaceSelector`可用于确定同步到哪个 Namespaces 目标，支持字段`matchLabels`。
请参阅[这里](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors)了解更多信息以及如何配置标签选择器。

如果`namespaceSelector`为空，捆绑目标将同步到所有命名空间。

---

## 安装

首先，将[cert-manager](https://cert-manager.io/docs/installation/)安装到集群，然后安装信任操作符。
建议在`cert-manager`命名空间中运行信任操作符。

```bash
helm repo add jetstack https://charts.jetstack.io --force-update
helm upgrade -i -n cert-manager cert-manager jetstack/cert-manager --set installCRDs=true --wait --create-namespace
helm upgrade -i -n cert-manager trust-manager jetstack/trust-manager --wait
```

#### 快速入门示例

```bash
kubectl create -n cert-manager configmap source-1 --from-literal=cm-key=123
kubectl create -n cert-manager secret generic source-2 --from-literal=sec-key=ABC
kubectl apply -f - <<EOF
apiVersion: trust.cert-manager.io/v1alpha1
kind: Bundle
metadata:
  name: example-bundle
spec:
  sources:
  - configMap:
      name: "source-1"
      key: "cm-key"
  - secret:
      name: "source-2"
      key: "sec-key"
  - inLine: |
      hello world!
  target:
    configMap:
      key: "target-key"
EOF
```

```bash
kubectl get bundle
NAME             TARGET       SYNCED   REASON   AGE
example-bundle   target-key   True     Synced   5s
```

```bash
kubectl get cm -A --field-selector=metadata.name=example-bundle
NAMESPACE            NAME             DATA   AGE
cert-manager         example-bundle   1      2m18s
default              example-bundle   1      2m18s
kube-node-lease      example-bundle   1      2m18s
kube-public          example-bundle   1      2m18s
kube-system          example-bundle   1      2m18s
local-path-storage   example-bundle   1      2m18s
```

```bash
kubectl get cm -n kube-system example-bundle -o jsonpath="{.data['target-key']}"
123
ABC
hello world!
```
