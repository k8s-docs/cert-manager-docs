---
title: "配置 HTTP01 入口提供商"
linkTitle: "HTTP01"
weight: 10
type: "docs"
---

此页面包含的不同选项的详细信息可在`Issuer`资源的 HTTP01 挑战求解器配置.
有关配置`ACME`发行人及其 API 格式的更多信息, 读出的[ACME 发行](../)文档.

你可以阅读有关如何 HTTP01 挑战类型的作品 在[Let's Encrypt 挑战类型页](https://letsencrypt.org/docs/challenge-types/#http-01-challenge).

下面是一个简单的例子`HTTP01` `ACME` 发行者 与配置更多的选择 下面:

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: example-issuer
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: example-issuer-account-key
    solvers:
      - http01:
          ingress:
            class: nginx
```

## 选项

该 HTTP01 发行人支持许多其他选项.
有关的可供选择的范围内的全部细节, 读出的[参考文档](../../../reference/api-docs/#acme.cert-manager.io/v1alpha2.ACMEChallengeSolverHTTP01).

### `ingressClass`

如果指定了`ingressClass`字段, 证书管理器将创建新的`Ingress`资源为了流量路由到`acmesolver` pods, 这是负责应对挑战 ACME 验证请求.

如果没有指定此字段, 和也没有指定`ingressName`, 证书管理器将默认为创建新的`Ingress`资源 但 **不** 设置对这些资源入口类, 这意味着安装集群中的 _所有_ 入口控制器将服务流量的挑战求解, 潜在地发生的额外费用.

### `ingressName`

如果指定了`ingressName`字段, 证书管理器将为了解决 HTTP01 挑战编辑命名的入口资源.

这对于具有入口控制器兼容性有用如 `ingress-gce`, 其利用所创建的每个`Ingress`资源的唯一的 IP 地址.

应使用入口控制器，暴露出一个 IP 对所有入口资源时，因为它可以产生具有某些入口控制器具体注解的相容性问题，可以避免此模式。

### `servicePort`

在极少数情况下，它可能是不可能的/需要使用`NodePort`类型的 HTTP01 质询响应服务， 例如因为 Kubernetes 跌停限制.
定义询问响应过程中要使用的 Kubernetes 服务类型指定以下 HTTP01 配置:

```yaml
http01:
  # Valid values are ClusterIP and NodePort
  serviceType: ClusterIP
```

默认情况下，当你没有设置 HTTP01 或当你设置`serviceType`为空字符串做类型`NodePort`将使用.
一般情况下没有必要更改此.

### `podTemplate`

您可能希望更改或添加标签和求解吊舱的注解.
这些可以在`metadata`场下下`podTemplate`被配置.

同样，你可以通过`podTemplate`的`spec`场下的配置设置`nodeSelector`，tolerations 和求解吊舱的亲和力.
没有其他规格的字段可以编辑.

如何可以配置模板的一个例子是这样:

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: ...
spec:
  acme:
    server: ...
    privateKeySecretRef:
      name: ...
    solvers:
      - http01:
          ingress:
            podTemplate:
              metadata:
                labels:
                  foo: "bar"
                  env: "prod"
              spec:
                nodeSelector:
                  bar: baz
```

所添加的标签和注释将合并在证书管理器默认的顶部，覆盖项使用相同的密钥.

在`podTemplate`存在的任何其他领域.

### `ingressTemplate`

It 是可能的标签和注释添加到解算入资源.
这些可以在`metadata`场下下`ingressTemplate`被配置:

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: ...
spec:
  acme:
    server: ...
    privateKeySecretRef:
      name: ...
    solvers:
      - http01:
          ingress:
            ingressTemplate:
              metadata:
                labels:
                  foo: "bar"
                annotations:
                  "nginx.ingress.kubernetes.io/whitelist-source-range": "0.0.0.0/0,::/0"
                  "nginx.org/mergeable-ingress-type": "minion"
                  "traefik.ingress.kubernetes.io/frontend-entry-points": "http"
```

所添加的标签和注释将合并在证书管理器默认的顶部，覆盖项使用相同的密钥.

入口的任何其他字段可编辑.
