# Webhook

ACME DNS-01 挑战使用外部 Webhook 求解器

webhook `Issuer`是一个通用的 ACME 求解器。
实际工作由外部服务完成。
查看[`dns-providers`](../../../contributing/dns-providers.md)各自的文档。

查看更多 webhook 求解器<https://github.com/topics/cert-manager-webhook>。

下面是如何配置 webhook 提供者的一个例子。
所有`DNS01`提供程序将包含他们自己的特定配置，但都需要一个`groupName` 和 `solverName`字段。

```yaml
apiVersion: cert-manager.io/v1
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
