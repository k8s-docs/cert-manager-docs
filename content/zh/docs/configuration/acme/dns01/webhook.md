---
title: "Webhook"
linkTitle: "Webhook"
weight: 30
type: "docs"
---

webhook `Issuer`是一个通用的 ACME 求解.
实际工作是由外部服务完成.
看的[`dns-providers`](../../../../contributing/dns-providers/)各自的文件.

查看更多 webhook 求解器在 https://github.com/topics/cert-manager-webhook.

下面是一个 webhook 提供商如何被配置的例子.
所有`DNS01`供应商将包含自己的特定配置但都需要一个'groupName`和`solverName`字段.

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: example-issuer
spec:
  acme:
   ...
    solvers:
    - dns01:
        webhook:
          groupName: $WEBHOOK_GROUP_NAME
          solverName: $WEBHOOK_SOLVER_NAME
          config:
            ...
            <webhook-specific-configuration>
```
