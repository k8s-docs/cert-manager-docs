---
title: "Vault"
linkTitle: ""
weight: 30
type: "docs"
---

> vault 是一个密码/证书集中式管理工具，通过 HTTP-API 对外提供统一的密码访问入口，并且提供权限控制以及详细的日志审计功能。
> 一个系统可能需要访问多个带密码的后端：例如数据库、通过 API keys 对外部系统进行调用，面向服务的架构通信等等。

`Vault` `Issuer` 代表的证书颁发机构 [Vault](https://www.vaultproject.io/) - 一机多用 secret store 可用于签收公钥基础结构证书 (PKI).
Vault 是一个外部项目证书管理器，因此, 本指南假设已经配置和正确部署, 准备签约.
你可以阅读更多关于如何配置 Vault 权威证书 [这里](https://www.vaultproject.io/docs/secrets/pki/).

此`Issuer`类型通常用于 当 Vault 已经被你的基础设施中使用,或者你想利用它的特性集 其中 CA 发行人不能单独提供.

## 部署

All Vault issuers share common configuration for requesting certificates,
namely the server, path, and CA bundle:

- Server is the URL whereby Vault is reachable.
- Path is the Vault path that will be used for signing. Note that the path
  _must_ use the `sign` endpoint.
- CA bundle denotes an optional field containing a base64 encoded string of the
  Certificate Authority to trust the Vault connection. This is typically
  _always_ required when using an `https` URL.

Below is an example of a configuration to connect a Vault server.

> **Warning**: This configuration is incomplete as no authentication methods have
> been added.

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: vault-issuer
  namespace: sandbox
spec:
  vault:
    path: pki_int/sign/example-dot-com
    server: https://vault.local
    caBundle: <base64 encoded CA Bundle PEM file>
    auth: ...
```

## 认证

In order to request signing of certificates by Vault, the issuer must be able to
properly authenticate against it. cert-manger provides multiple approaches to
authenticating to Vault which are detailed below.

### 通过 AppRole 认证

An [AppRole](https://www.vaultproject.io/docs/auth/approle.html) is a method of
authenticating to Vault through use of it's internal role policy system. This
authentication method requires that the issuer has possession of the `SecretID`
secret key, the `RoleID` of the role to assume, and the app role path. Firstly,
the secret ID key must be stored within a Kubernetes `Secret` that resides in the
same namespace as the `Issuer`, or otherwise inside the `Cluster Resource Namespace` in the case of a `ClusterIssuer`.

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: cert-manager-vault-approle
  namespace: sandbox
data:
  secretId: "MDI..."
```

Once the `Secret` has been created, the `Issuer` is ready to be deployed which
references this `Secret`, as well as the data key of the field that stores the
secret ID.

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: vault-issuer
  namespace: sandbox
spec:
  vault:
    path: pki_int/sign/example-dot-com
    server: https://vault.local
    caBundle: <base64 encoded caBundle PEM file>
    auth:
      appRole:
        path: approle
        roleId: "291b9d21-8ff5-..."
        secretRef:
          name: cert-manager-vault-approle
          key: secretId
```

### 用记号验证

This method of authentication uses a token string that has been generated from
one of the many authentication backends that Vault supports. These tokens have
an expiry and so need to be periodically refreshed. You can read more on Vault
tokens [here](https://www.vaultproject.io/docs/concepts/tokens.html).

> **Note**: cert-manager does not refresh these token automatically and so another
> process must be put in place to do this.

Firstly, the token is be stored inside a Kubernetes `Secret` inside the same
namespace as the `Issuer` or otherwise in the `Cluster Resource Namespace` in
the case of using a `ClusterIssuer`.

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: cert-manager-vault-token
  namespace: sandbox
data:
  token: "MjI..."
```

Once submitted, the Vault issuer is able to be created using token
authentication by referencing this `Secret` along with the key of the field the
token data is stored at.

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: vault-issuer
  namespace: sandbox
spec:
  vault:
    path: pki_int/sign/example-dot-com
    server: https://vault.local
    caBundle: <base64 encoded caBundle PEM file>
    auth:
      tokenSecretRef:
        name: cert-manager-vault-token
        key: token
```

### 与 Kubernetes 服务帐户验证

Vault can be configured so that applications can authenticate using Kubernetes
[`Service Account Tokens`](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin).
You find documentation on how to configure Vault to authenticate using Service
Account Tokens [here](https://www.vaultproject.io/docs/auth/kubernetes.html).

For the Vault issuer to use this authentication, cert-manager must get access to
the token that is stored in a Kubernetes `Secret`. Kubernetes Service Account
Tokens are already stored in `Secret` resources so this does not need to be
created manually however, you must ensure that it is present in the same
namespace as the `Issuer`, or otherwise in the `Cluster Resource Namespace` in
the case of using a `ClusterIssuer`.

This authentication method also expects a `role` field which is the Vault role
that the Service Account is to assume, as well as an optional `path` field which
is the authentication mount path, defaulting to `kubernetes`.

The following example will be making use of the Service Account
`my-service-account`. The secret data field key will be `token` if the `Secret`
has been created by Kubernetes.

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: vault-issuer
  namespace: sandbox
spec:
  vault:
    path: pki_int/sign/example-dot-com
    server: https://vault.local
    caBundle: <base64 encoded caBundle PEM file>
    auth:
      kubernetes:
        role: my-app-1
        mountPath: /v1/auth/kubernetes
        secretRef:
          name: my-service-account-token-hvwsb
          key: token
```

## 验证发行人部署

Once the Vault issuer has been deployed, it will be marked as ready if the
configuration is valid. Replace `issuers` here with `clusterissuers` if that is what has
been deployed.

```bash
$ kubectl get issuers vault-issuer -n sandbox -o wide
NAME          READY   STATUS          AGE
vault-issuer  True    Vault verified  2m
```

Certificates are now ready to be requested by using the Vault issuer named
`vault-issuer` within the `sandbox` namespace.
