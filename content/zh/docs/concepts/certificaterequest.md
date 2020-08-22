---
title: "证书请求(CertificateRequest)"
linkTitle: "证书请求"
weight: 300
type: "docs"
---

该`CertificateRequest`是`cert-manager`一个名称空间资源是用于从[`Issuer`](../issuer/)请求 X509 证书.
资源含有 PEM 的 base64 编码字符串编码证书请求其被发送到所引用的发行者.
一个成功的发行将返回一个签名证书, 基于证书签名请求.
`CertificateRequests`通常由控制器或其他系统消耗和管理而不应被人类使用 - 除非特别需要.

一个简单的`CertificateRequest`如下所示:

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: CertificateRequest
metadata:
  name: my-ca-cr
spec:
  csr: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQzNqQ0NBY1lDQVFBd2daZ3hDekFKQmdOVkJBWVRBbHBhTVE4d0RRWURWUVFJREFaQmNHOXNiRzh4RFRBTApCZ05WQkFjTUJFMXZiMjR4RVRBUEJnTlZCQW9NQ0VwbGRITjBZV05yTVJVd0V3WURWUVFMREF4alpYSjBMVzFoCmJtRm5aWEl4RVRBUEJnTlZCQU1NQ0dwdmMyaDJZVzVzTVN3d0tnWUpLb1pJaHZjTkFRa0JGaDFxYjNOb2RXRXUKZG1GdWJHVmxkWGRsYmtCcVpYUnpkR0ZqYXk1cGJ6Q0NBU0l3RFFZSktvWklodmNOQVFFQkJRQURnZ0VQQURDQwpBUW9DZ2dFQkFLd01tTFhuQkNiRStZdTIvMlFtRGsxalRWQ3BvbHU3TlZmQlVFUWl1bDhFMHI2NFBLcDRZQ0c5Cmx2N2kwOHdFMEdJQUgydnJRQmxVd3p6ZW1SUWZ4YmQvYVNybzRHNUFBYTJsY2NMaFpqUlh2NEVMaER0aVg4N3IKaTQ0MWJ2Y01OM0ZPTlRuczJhRkJYcllLWGxpNG4rc0RzTEVuZmpWdXRiV01Zeis3M3ptaGZzclRJUjRzTXo3cQpmSzM2WFM4UkRjNW5oVVcyYU9BZ3lnbFZSOVVXRkxXNjNXYXVhcHg2QUpBR1RoZnJYdVVHZXlZUUVBSENxZmZmCjhyOEt3YTFYK1NwYm9YK1ppSVE0Nk5jQ043OFZnL2dQVHNLZmphZURoNWcyNlk1dEVidHd3MWdRbWlhK0MyRHIKWHpYNU13RzJGNHN0cG5kUnRQckZrU1VnMW1zd0xuc0NBd0VBQWFBQU1BMEdDU3FHU0liM0RRRUJDd1VBQTRJQgpBUUFXR0JuRnhaZ0gzd0N3TG5IQ0xjb0l5RHJrMUVvYkRjN3BJK1VVWEJIS2JBWk9IWEFhaGJ5RFFLL2RuTHN3CjJkZ0J3bmlJR3kxNElwQlNxaDBJUE03eHk5WjI4VW9oR3piN0FVakRJWHlNdmkvYTJyTVhjWjI1d1NVQmxGc28Kd005dE1QU2JwcEVvRERsa3NsOUIwT1BPdkFyQ0NKNnZGaU1UbS9wMUJIUWJSOExNQW53U0lUYVVNSFByRzJVMgpjTjEvRGNMWjZ2enEyeENjYVoxemh2bzBpY1VIUm9UWmV1ZEp6MkxmR0VHM1VOb2ppbXpBNUZHd0RhS3BySWp3ClVkd1JmZWZ1T29MT1dNVnFNbGRBcTlyT24wNHJaT3Jnak1HSE9tTWxleVdPS1AySllhaDNrVDdKU01zTHhYcFYKV0ExQjRsLzFFQkhWeGlKQi9Zby9JQWVsCi0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=
  isCA: false
  keyUsages:
    - signing
    - digital signature
    - server auth
  duration: 90d
  issuerRef:
    name: ca-issuer
    # We can reference ClusterIssuers by changing the kind here.
    # The default value is Issuer (i.e. a locally namespaced Issuer)
    kind: Issuer
    group: cert-manager.io
```

这`CertificateRequest`将使`cert-manager`试图请求默认的发行人组 `cert-manager.io`里`Issuer` `ca-issuer` , 返回基于证书签名请求的证书.
可以指定`issuerRef`里面的其他组这将改变针对发行人其他外部, 你可能已经安装了第三方发行商.

资源也暴露了陈述证书 CA 选项, 密钥用法, 并要求有效期限.

一个成功的发行证书签名请求将导致更新资源, 设置与签名证书的状态, 证书的 CA (如果可供使用的话), 并设置`Ready`条件`True`;.

无论是签发证书签名请求的成功与否, 发行重试 _不_ 会发生. 这是一些其他的控制器负责管理`CertificateRequests`的逻辑和生命周期.

## Conditions

`CertificateRequests`有一组强定义的条件应使用和由控制器或服务依靠做出什么行动下一步采取的资源决定.
每个条件由一对`Ready`的 - 一个布尔值, 和 `Reason` - 字符串.
该组值和含义如下所示:

| Ready | Reason  | Condition Meaning                                                                                                                        |
| ----- | ------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| False | Pending | 该`CertificateRequest`目前正在等待, 等待一些其他操作发生. 这可能是因为`Issuer`尚不存在或`Issuer`是在签发证书的过程.                      |
| False | Failed  | 证书未能发行 - 无论是返回的证书无法被解码或者用于签名引用发行人的实例失败. 没有进一步的行动将会在`CertificateRequest`被它的控制器应采取. |
| True  | Issued  | 签名证书已被引用`Issuer`已成功发行.                                                                                                      |
