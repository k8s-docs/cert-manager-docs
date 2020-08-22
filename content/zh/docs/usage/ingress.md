---
title: "保护入口资源"
linkTitle: ""
weight: 100
type: "docs"
description: >
  一个常见的用例的证书管理器是请求TLS签名证书以保护您的入口资源.
---

这可以通过简单地添加注释到你的`Ingress`资源来完成 和证书管理器将有助于为您创建`Certificate`资源.
一个小的 cert-manager, ingress-shim 的子组件 , 负责本.

## 这个怎么运作

子组件 ingress-shim 观测 `Ingress` 资源 在您的集群.
如果观察到的 `Ingress` 在[支持的注解](#supported-annotations)部分中描述的注释 , 这样既保证了`Certificate`资源在所提供的名称 `tls.secretName` 字段 并配置为在`Ingress`描述存在. 例如:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    # add an annotation indicating the issuer to use.
    cert-manager.io/cluster-issuer: nameOfClusterIssuer
  name: myIngress
  namespace: myIngress
spec:
  rules:
    - host: myingress.com
      http:
        paths:
          - backend:
              serviceName: myservice
              servicePort: 80
            path: /
  tls: # < placing a host in the TLS config will indicate a certificate should be created
    - hosts:
        - myingress.com
      secretName: myingress-cert # < cert-manager will store the created certificate in this secret.
```

## 支持的注解

您可以以指定的`Ingress`资源以下注释来触发`Certificate`资源来自动创建:

- `cert-manager.io/issuer`: 一个`Issuer`的名字获取此`Ingress`要求的证书. 发行者必须在相同的命名空间`Ingress`资源.

- `cert-manager.io/cluster-issuer`: 一个`ClusterIssuer`的名字来获取这个`Ingress`要求的证书. 不要紧，它的命名空间`Ingress`所在,作为`ClusterIssuers`是非名称空间资源.

- `cert-manager.io/issuer-kind`: 外部`Issuer`控制器的`CustomResourceDefinition`的名字 (仅需要外的树`Issuers`)

- `cert-manager.io/issuer-group`: 外部`Issuer`控制器的 API 组的名称 (仅需要外的树`Issuers`)

- `kubernetes.io/tls-acme: "true"`: 这个注释需要 ingress-shim 的其他配置[见下文](./#optional-configuration).
  亦即, 默认`Issuer`必须被指定作为参数传递给 ingress-shim 容器.

- `acme.cert-manager.io/http01-ingress-class`: 这个注解允许您配置入口类 将用于解决这个入口挑战.
  定制这个当你试图以保护内部服务是非常有用的, 和需要使用不同的入口类解决的挑战该入口的.
  如果没有指定和 `acme-http01-edit-in-place` 注释未设置, 此默认为入口类的入口资源.

- `acme.cert-manager.io/http01-edit-in-place: "true"`: this controls whether the
  ingress is modified 'in-place', or a new one is created specifically for the
  HTTP01 challenge. If present, and set to "true", the existing ingress will be
  modified. Any other value, or the absence of the annotation assumes "false".
  This annotation will also add the annotation
  `"cert-manager.io/issue-temporary-certificate": "true"` onto created
  certificates which will cause a [temporary certificate](../certificate/#temporary-certificates-whilst-issuing)
  to be set on the resulting `Secret` until the final signed certificate has been
  returned. This is useful for keeping compatibility with the `ingress-gce`
  component.

## 可选配置

The ingress-shim sub-component is deployed automatically as part of
installation.

If you would like to use the old
[kube-lego](https://github.com/jetstack/kube-lego) `kubernetes.io/tls-acme: "true"` annotation for fully automated TLS, you will need to configure a default
`Issuer` when deploying cert-manager. This can be done by adding the following
`--set` when deploying using Helm:

```bash
   --set ingressShim.defaultIssuerName=letsencrypt-prod \
   --set ingressShim.defaultIssuerKind=ClusterIssuer \
   --set ingressShim.defaultIssuerGroup=cert-manager.io
```

Or by adding the following arguments to the cert-manager deployment
`podTemplate` container arguments.

```
  - --default-issuer-name=letsencrypt-prod
  - --default-issuer-kind=ClusterIssuer
  - --default-issuer-group=cert-manager.io
```

In the above example, cert-manager will create `Certificate` resources that
reference the `ClusterIssuer` `letsencrypt-prod` for all Ingresses that have a
`kubernetes.io/tls-acme: "true"` annotation.

For more information on deploying cert-manager, read the [installation
guide](../../installation/).

## 故障排除

If you do not see a `Certificate` resource being created after applying the ingress-shim annotations check that at least `cert-manager.io/issuer` or `cert-manager.io/cluster-issuer` is set. If you want to use `kubernetes.io/tls-acme: "true"` make sure to have checked all steps above and you might want to look for errors in the cert-manager pod logs if not resolved.
