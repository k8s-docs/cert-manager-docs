---
title: "群集资源命名空间"
linkTitle: "命名空间"
weight: 60
type: "docs"
---

该`ClusterIssuer`资源是集群范围的.
这意味着，参考通过`secretName`字段`secret`时, `secrets` 将在`Cluster Resource Namespace`里被寻找.
默认, 这个命名空间是 `cert-manager` 然而可以通过一个标签 cert-manager-controller 部件来改变:

```bash
--cluster-resource-namespace=my-namespace
```
