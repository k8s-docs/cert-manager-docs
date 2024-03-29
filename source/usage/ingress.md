---
title: Securing Ingress Resources
description: "cert-manager usage: Kubernetes Ingress"
---

# 保护入口源

证书管理器的一个常见用例是请求 TLS 签名证书来保护您的入口源。
这可以通过简单地添加注释到您的`Ingress`源和证书管理器将促进为您创建`Certificate`源来完成。
cert-manager 的一个小子组件 ingress-shim 负责这一点。

## 工作原理

子组件 ingress-shim 监视整个集群中的`Ingress`源。
如果它观察到一个带有[受支持的注释](#supported-annotations)部分中描述的注释的`Ingress`，它将确保在`Ingress`的命名空间中存在一个`Certificate`源，其名称在`tls.secretName`字段中提供，并按照`Ingress`中描述的配置。
例如:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    # add an annotation indicating the issuer to use.
    cert-manager.io/cluster-issuer: nameOfClusterIssuer
  name: myIngress
  namespace: myIngress
spec:
  rules:
    - host: example.com
      http:
        paths:
          - pathType: Prefix
            path: /
            backend:
              service:
                name: myservice
                port:
                  number: 80
  tls: # < placing a host in the TLS config will determine what ends up in the cert's subjectAltNames
    - hosts:
        - example.com
      secretName: myingress-cert # < cert-manager will store the created certificate in this secret.
```

## 支持注释

您可以在 Ingress 源上指定以下注释，以触发自动创建 Certificate 源:

- `cert-manager.io/issuer`: Issuer 的名称，以获得此进入所需的证书。Issuer _必须_ 与 Ingress 源在相同的命名空间中。

- `cert-manager.io/cluster-issuer`: ClusterIssuer 的名称，以获取此 Ingress 所需的证书。您的 Ingress 驻留在哪个名称空间并不重要，因为 ClusterIssuers 是非名称空间源。

- `cert-manager.io/issuer-kind`: 外部发行者源的类型，例如`AWSPCAIssuer`。这只对树外发行者有必要。

- `cert-manager.io/issuer-group`: the API group of the external issuer
  controller, for example `awspca.cert-manager.io`. This is only necessary for
  out-of-tree issuers.

- `kubernetes.io/tls-acme: "true"`: this annotation requires additional
  configuration of the ingress-shim [see below](#optional-configuration).
  Namely, a default Issuer must be specified as arguments to the ingress-shim
  container.

- `acme.cert-manager.io/http01-ingress-class`: this annotation allows you to
  configure the ingress class that will be used to solve challenges for this
  ingress. Customizing this is useful when you are trying to secure internal
  services, and need to solve challenges using a different ingress class to that
  of the ingress. If not specified and the `acme-http01-edit-in-place` annotation
  is not set, this defaults to the ingress class defined in the Issuer resource.

- `acme.cert-manager.io/http01-edit-in-place: "true"`: this controls whether the
  ingress is modified 'in-place', or a new one is created specifically for the
  HTTP01 challenge. If present, and set to "true", the existing ingress will be
  modified. Any other value, or the absence of the annotation assumes "false".
  This annotation will also add the annotation
  `"cert-manager.io/issue-temporary-certificate": "true"` onto created
  certificates which will cause a [temporary
  certificate](./certificate.md#temporary-certificates-whilst-issuing) to be set
  on the resulting Secret until the final signed certificate has been returned.
  This is useful for keeping compatibility with the `ingress-gce` component.

- `cert-manager.io/common-name`: (optional) this annotation allows you to
  configure `spec.commonName` for the Certificate to be generated.

- ` cert-manager.io/duration`: (optional) this annotation allows you to
  configure `spec.duration` field for the Certificate to be generated.

- `cert-manager.io/renew-before`: (optional) this annotation allows you to
  configure `spec.renewBefore` field for the Certificate to be generated.

- `cert-manager.io/usages`: (optional) this annotation allows you to configure
  `spec.usages` field for the Certificate to be generated. Pass a string with
  comma-separated values i.e "key agreement,digital signature, server auth"

- `cert-manager.io/revision-history-limit`: (optional) this annotation allows you to
  configure `spec.revisionHistoryLimit` field to limit the number of CertificateRequests to be kept for a Certificate.
  Minimum value is 1. If unset all CertificateRequests will be kept.

- `cert-manager.io/private-key-algorithm`: (optional) this annotation allows you to
  configure `spec.privateKey.algorithm` field to set the algorithm for private key generation for a Certificate.
  Valid values are `RSA`, `ECDSA` and `Ed25519`. If unset an algorithm `RSA` will be used.

- `cert-manager.io/private-key-encoding`: (optional) this annotation allows you to
  configure `spec.privateKey.encoding` field to set the encoding for private key generation for a Certificate.
  Valid values are `PKCS1` and `PKCS8`. If unset an algorithm `PKCS1` will be used.

- `cert-manager.io/private-key-size`: (optional) this annotation allows you to
  configure `spec.privateKey.size` field to set the size of the private key for a Certificate.
  If algorithm is set to `RSA`, valid values are `2048`, `4096` or `8192`, and will default to `2048` if not specified.
  If algorithm is set to `ECDSA`, valid values are `256`, `384` or `521`, and will default to `256` if not specified.
  If algorithm is set to `Ed25519`, size is ignored.

- `cert-manager.io/private-key-rotation-policy`: (optional) this annotation allows you to
  configure `spec.privateKey.rotationPolicy` field to set the rotation policy of the private key for a Certificate.
  Valid values are `Never` and `Always`. If unset a rotation policy `Never` will be used.

## 可选配置

ingress-shim 子组件作为安装的一部分自动部署。

If you would like to use the old
[kube-lego](https://github.com/jetstack/kube-lego) `kubernetes.io/tls-acme:
"true"` annotation for fully automated TLS, you will need to configure a default
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

Issuers configured via annotations have a preference over the default issuer. If a default issuer is configured via CLI flags and a `cert-manager.io/cluster-issuer` or `cert-manager.io/issuer` annotation also has been added to an Ingress, the created `Certificate` will refer to the issuer configured via annotation.

For more information on deploying cert-manager, read the [installation
guide](../installation/README.md).

## 故障排除

If you do not see a `Certificate` resource being created after applying the ingress-shim annotations check that at least `cert-manager.io/issuer` or `cert-manager.io/cluster-issuer` is set. If you want to use `kubernetes.io/tls-acme: "true"` make sure to have checked all steps above and you might want to look for errors in the cert-manager pod logs if not resolved.
