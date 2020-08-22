---
title: "使用Kubed跨命名空间同步Secret"
linkTitle: "同步秘密"
weight: 60
type: "docs"
---

它可能需要多个组件跨越命名空间消耗相同的`Secret` 已经由单一的`Certificate`创建.
要做到这一点，建议的方法是使用[kubed](https://github.com/appscode/kubed) [secret 同步功能](https://appscode.com/products/kubed/v0.11.0/guides/config-syncer/intra-cluster/).

为了使目标`Secret`要同步, 在创建证书之前`Secret`资源必须首先与正确标注创建, 否则`Secret`需要进行编辑，而不是.
下面示出了从`cert-manager`命名空间同步的证书的属于`sandbox`证书示例 ,进入`sandbox`命名空间.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: sandbox
  labels:
    cert-manager-tls: sandbox # Define namespace label for kubed
---
apiVersion: v1
data:
  ca.crt: ""
  tls.crt: ""
  tls.key: ""
kind: Secret
metadata:
  name: sandbox-tls
  namespace: cert-manager
  annotations:
    kubed.appscode.com/sync: "cert-manager-tls=sandbox" # Sync certificate to matching namespaces
type: kubernetes.io/tls
---
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: sandbox
  namespace: cert-manager
spec:
  secretName: sandbox-tls
  commonName: sandbox
  issuerRef:
    name: sandbox-ca
    kind: Issuer
    group: cert-manager.io
```
