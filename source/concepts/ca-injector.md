---
title: CA Injector
description: "cert-manager core concepts: CA Injector"
---

# CA Injector

`cainjector` 帮助配置 CA 证书: [Mutating Webhooks],[Validating Webhooks],[conversion webhooks] 和 [API Services]

特别是，`cainjector`填充了四种 API 类型的`caBundle`字段:`ValidatingWebhookConfiguration`,`MutatingWebhookConfiguration`,`CustomResourceDefinition` 和 `APIService`.

前三种源类型用于配置 Kubernetes API 服务器如何连接到 webhook。
这个`caBundle`数据由 Kubernetes API 服务器加载，用于验证 webhook API 服务器的服务证书。
`APIService`用于表示[扩展 API 服务器]。
`APIService`的`caBundle`可以填充 CA 证书，可用于验证 API 服务器的服务证书。

我们将这四种 API 类型称为 _injectable_ 源。

_injectable_ 源必须有以下注解之一:`cert-manager.io/inject-ca-from`,`cert-manager.io/inject-ca-from-secret`, 或 `cert-manager.io/inject-apiserver-ca`, 取决于注入 _源_。下面将更详细地解释这一点。

`cainjector` copies CA data from one of three _sources_:
a Kubernetes `Secret`,
a cert-manager `Certificate`, or from
the Kubernetes API server CA certificate (which `cainjector` itself uses to verify its TLS connection to the Kubernetes API server).

If the _source_ is a Kubernetes `Secret`, that resource MUST also have an `cert-manager.io/allow-direct-injection: "true"` annotation.
The three _source_ types are explained in more detail below.

## 举例

Here are examples demonstrating how to use the three `cainjector` _sources_.
In each case we use `ValidatingWebhookConfiguration` as the _injectable_,
but you can substitute `MutatingWebhookConfiguration` or `CustomResourceDefinition` definition instead.

### 从证书源中注入 CA 数据

Here is an example of a `ValidatingWebhookConfiguration`
configured with the annotation `cert-manager.io/inject-ca-from`,
which will make `cainjector` populate the `caBundle` field using CA data from a cert-manager `Certificate`.

NOTE: This example does not deploy a webhook server,
it only deploys a partial webhook configuration,
but it should be sufficient to help you understand what `cainjector` does:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: example1

---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: webhook1
  annotations:
    cert-manager.io/inject-ca-from: example1/webhook1-certificate
webhooks:
  - name: webhook1.example.com
    admissionReviewVersions:
      - v1
    clientConfig:
      service:
        name: webhook1
        namespace: example1
        path: /validate
        port: 443
    sideEffects: None

---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: webhook1-certificate
  namespace: example1
spec:
  secretName: webhook1-certificate
  dnsNames:
    - webhook1.example1
  issuerRef:
    name: selfsigned

---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: selfsigned
  namespace: example1
spec:
  selfSigned: {}
```

You should find that the `caBundle` value is now identical to the CA value in the `Secret` for the `Certificate`:

```
kubectl get validatingwebhookconfigurations.admissionregistration.k8s.io webhook1 -o yaml | grep caBundle
kubectl -n example1 get secret webhook1-certificate -o yaml | grep ca.crt
```

And after a short time, the Kubernetes API server will read that new `caBundle` value and use it to verify a TLS connection to the webhook server.

### 从 Secret 源注入 CA 数据

Here is another example of a `ValidatingWebhookConfiguration`
this time configured with the annotation `cert-manager.io/inject-ca-from-secret`,
which will make `cainjector` populate the `caBundle` field using CA data from a Kubernetes `Secret`.

NOTE: This example does not deploy a webhook server,
it only deploys a partial webhook configuration,
but it should be sufficient to help you understand what `cainjector` does:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: example2

---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: webhook2
  annotations:
    cert-manager.io/inject-ca-from-secret: example2/example-ca
webhooks:
  - name: webhook2.example.com
    admissionReviewVersions:
      - v1
    clientConfig:
      service:
        name: webhook2
        namespace: example2
        path: /validate
        port: 443
    sideEffects: None

---
apiVersion: v1
kind: Secret
metadata:
  name: example-ca
  namespace: example2
  annotations:
    cert-manager.io/allow-direct-injection: "true"
type: kubernetes.io/tls
data:
  ca.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM5akNDQWQ2Z0F3SUJBZ0lRTkdJZ24yM3BQYVpNbk9MUjJnVmZHakFOQmdrcWhraUc5dzBCQVFzRkFEQVYKTVJNd0VRWURWUVFERXdwRmVHRnRjR3hsSUVOQk1CNFhEVEl3TURreU5ERTFOREEwTVZvWERUSXdNVEl5TXpFMQpOREEwTVZvd0ZURVRNQkVHQTFVRUF4TUtSWGhoYlhCc1pTQkRRVENDQVNJd0RRWUpLb1pJaHZjTkFRRUJCUUFECmdnRVBBRENDQVFvQ2dnRUJBS2F3RzVoMzlreHdyNEl0WCtHaDNYVWQrdTVJc2ZlSFdoTTc4TTRQTmZFeXhQMXoKRmNLN1d0MHJFMkwwNUppYmQ4ZjNpb3k5OXNnQ3I4OEw2SWxYZTB0RnkzNysxenJ4TFluR2hDQnZzZjltd0hLbgpIVTEvNERwQjROZkhPbFllNE9tbHVoNE9HdmZINU1EbDh5OWZGMjhXRXVBQ2dwdmpCUWxvRDNlVjJ5UmJvQ2kyCmtSTDJWYTFZL0FQZEpWK21VYkFvZmg0bllmUmNLRTJsSUg0RG5ZdXFPU3JaaituZUQ2M2RTSktxcHQ5K2luN2YKNHljZ2pQYU93MmdyKzhLK291QTlSQTV1VDI3SVNJcUJDcEV6elRqbVBUUWNvUTYxZGF0aDZkc1lsTEU4aWZWUwp4RWZuVEdQKy94M0FXQXR4eU5lanVuZGFXbVNFL3h5OHh0K0FxblVDQXdFQUFhTkNNRUF3RGdZRFZSMFBBUUgvCkJBUURBZ0trTUE4R0ExVWRFd0VCL3dRRk1BTUJBZjh3SFFZRFZSME9CQllFRkowNkc5eEc2V1VBTHB6T3JYaHAKV2dsTm5qMkFNQTBHQ1NxR1NJYjNEUUVCQ3dVQUE0SUJBUUI3ZG9CZnBLR3o4VlRQSnc0YXhpdisybzJpMHE1SQpSRzU2UE81WnhKQktZQlRROElHQmFOSm1yeGtmNTJCV0ttUGp4cXlNSGRwWjVBU00zOUJkZVUzRGtEWHp4RkgwCjM5RU12UnhIUERyMGQ4cTFFbndQT0xZY1hzNjJhYjdidE11cTJUMFNNZzRYMkY5VmNKTW5YdjlrNnA0VGZNR3MKVThCQnJhVGhUZm53ejBsWXMyblFjdzNmZjZ1bG1wWlk4K3BTak1aVDNJZHZOMFA4Y2hOdUlmUFRHWDJmSlo2NQpxcUUrelRoU3hIeXFTOTVoczhsd1lRRUhGQlVsalRnMCtQZThXL0hOSXZBOU9TYWw1U3UvdlhydmcxN04xdHVyCk5XcWRyZU5OVm1ubXMvTFJodmthWTBGblRvbFNBRkNXWS9GSDY5ZzRPcThiMHVyK3JVMHZOZFFXCi0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
  tls.key: ""
  tls.crt: ""
```

You should find that the `caBundle` value is now identical to the `ca.crt` value in the `Secret`:

```
kubectl get validatingwebhookconfigurations.admissionregistration.k8s.io webhook2  -o yaml | grep caBundle
```

And after a short time, the Kubernetes API server will read that new `caBundle` value and use it to verify a TLS connection to the webhook server.

This `Secret` based injection mechanism can operate independently of the `Certificate` based mechanism described earlier.
It will work without the cert-manager CRDs installed
and it will work if the cert-manager CRDs and associated webhook servers are not yet configured.

NOTE: For this reason, cert-manager uses the `Secret` based injection mechanism to bootstrap its own webhook server.
The cert-manager webhook server generates its own private key and self-signed certificate and places them in a `Secret` when it starts up.

### 注入 Kubernetes API 服务 CA

Here is another example of a `ValidatingWebhookConfiguration`
this time configured with the annotation `cert-manager.io/inject-apiserver-ca: "true"`,
which will make `cainjector` populate the `caBundle` field using the same CA certificate used by the Kubernetes API server.

NOTE: This example does not deploy a webhook server,
it only deploys a partial webhook configuration,
but it should be sufficient to help you understand what `cainjector` does:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: example3

---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: webhook3
  annotations:
    cert-manager.io/inject-apiserver-ca: "true"
webhooks:
  - name: webhook3.example.com
    admissionReviewVersions:
      - v1
    clientConfig:
      service:
        name: webhook3
        namespace: example3
        path: /validate
        port: 443
    sideEffects: None
```

You should find that the `caBundle` value is now identical to the CA used in your `KubeConfig` file:

```
kubectl get validatingwebhookconfigurations.admissionregistration.k8s.io webhook3 -o yaml | grep caBundle
kubectl config  view --minify --raw | grep certificate-authority-data
```

And after a short time, the Kubernetes API server will read that new `caBundle` value and use it to verify a TLS connection to the webhook server.

NOTE: In this case you will have to ensure that your webhook is configured to serve a TLS certificate that has been signed by the Kubernetes cluster CA.
The disadvantages of this mechanism are that: you will require access to the private key of the Kubernetes cluster CA and you will need to manually rotate the webhook certificate.

[validating webhooks]: https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#validatingadmissionwebhook
[mutating webhooks]: https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook
[conversion webhooks]: https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/#webhook-conversion
[api services]: https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/api-service-v1/
[extension api server]: https://kubernetes.io/docs/tasks/extend-kubernetes/setup-extension-api-server/
