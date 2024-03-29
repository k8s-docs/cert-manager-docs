---
title: Certificate Resources
description: "cert-manager usage: Certificates"
---

# 证书源

在证书管理器中，[`Certificate`](../concepts/certificate.md)源表示证书请求的可读定义，该证书请求将由颁发者执行，并保持最新。
这是与证书管理器交互以请求已签名证书的常用方式。

为了颁发任何证书，您需要首先配置一个[`Issuer`](../configuration/README.md) 或 [`ClusterIssuer`](../configuration/README.md) 源。

## 创建证书源

`Certificate` 源指定用于生成证书签名请求的字段，然后由您引用的颁发者类型完成。
`Certificate` 通过指定`certificatespecissuerRef`字段指定他们想从哪个颁发者获得证书。

下面是一个`Certificate`源，用于`example.com` 和 `www.example.com`DNS 名称，`spiffe://cluster.local/ns/sandbox/sa/example` URI 主题替代名称，有效期为 90 天，并在到期前 15 天更新。
它包含了一个`Certificate`源可能拥有的所有选项的详尽列表，但只有一个字段的子集是必需的。

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com
  namespace: sandbox
spec:
  # 总是需要使用秘密名称。
  secretName: example-com-tls

  # secretTemplate是可选参数。
  # 如果设置了，这些注释和标签将被复制到名为example-com-tls的Secret。
  # 如果证书的secretTemplate发生变化，这些标签和注释将重新协调。
  # secretTemplate也是强制的，因此第三方对Secret的相关标签和注释更改将被cert-manager覆盖以匹配secretTemplate。
  secretTemplate:
    annotations:
      my-secret-annotation-1: "foo"
      my-secret-annotation-2: "bar"
    labels:
      my-secret-label: foo

  duration: 2160h # 90d
  renewBefore: 360h # 15d
  subject:
    organizations:
      - jetstack
  # 自2000年以来，通用名称字段的使用已被弃用，不鼓励使用。
  commonName: example.com
  isCA: false
  privateKey:
    algorithm: RSA
    encoding: PKCS1
    size: 2048
  usages:
    - server auth
    - client auth
  # 至少需要DNS名称、URI或IP地址中的一个。
  dnsNames:
    - example.com
    - www.example.com
  uris:
    - spiffe://cluster.local/ns/sandbox/sa/example
  ipAddresses:
    - 192.168.0.5
  # 始终需要发行者引用。
  issuerRef:
    name: ca-issuer
    # 我们可以通过更改这里的类型来引用 ClusterIssuers。
    # 默认值是Issuer(即本地命名空间的Issuer)。
    kind: Issuer
    # 这是可选的，因为cert-manager将默认为此值，但如果您使用外部颁发者，请将此更改为该颁发者组。
    group: cert-manager.io
```

The signed certificate will be stored in a `Secret` resource named
`example-com-tls` in the same namespace as the `Certificate` once the issuer has
successfully issued the requested certificate.

If `secretTemplate` is present, annotations and labels set in this property
will be copied over to `example-com-tls` secret. Both properties are optional.

The `Certificate` will be issued using the issuer named `ca-issuer` in the
`sandbox` namespace (the same namespace as the `Certificate` resource).

> Note: If you want to create an `Issuer` that can be referenced by
> `Certificate` resources in _all_ namespaces, you should create a
> [`ClusterIssuer`](../concepts/issuer.md#namespaces) resource and set the
> `certificate.spec.issuerRef.kind` field to `ClusterIssuer`.

> Note: The `renewBefore` and `duration` fields must be specified using a [Go
> `time.Duration`](https://golang.org/pkg/time/#ParseDuration) string format,
> which does not allow the `d` (days) suffix. You must specify these values
> using `s`, `m`, and `h` suffixes instead. Failing to do so without installing
> the [`webhook component`](../concepts/webhook.md) can prevent cert-manager
> from functioning correctly
> [`#1269`](https://github.com/cert-manager/cert-manager/issues/1269).

> Note: Take care when setting the `renewBefore` field to be very close to the
> `duration` as this can lead to a renewal loop, where the `Certificate` is always
> in the renewal period. Some `Issuers` set the `notBefore` field on their
> issued X.509 certificates before the issue time to fix clock-skew issues,
> leading to the working duration of a certificate to be less than the full
> duration of the certificate. For example, Let's Encrypt sets it to be one hour
> before issue time, so the actual _working duration_ of the certificate is 89
> days, 23 hours (the _full duration_ remains 90 days).

A full list of the fields supported on the Certificate resource can be found in
the [API reference documentation](../reference/api-docs.md#cert-manager.io/v1.CertificateSpec).

<h2 id="key-usages">X.509 key usages and extended key usages</h2>

cert-manager supports requesting certificates that have a number of [custom key
usages](https://tools.ietf.org/html/rfc5280#section-4.2.1.3) and [extended key
usages](https://tools.ietf.org/html/rfc5280#section-4.2.1.12). Although
cert-manager will attempt to honor this request, some issuers will remove, add
defaults, or otherwise completely ignore the request.
The `CA` and `SelfSigned` `Issuer` will always return certificates matching the usages you have requested.

Unless any number of usages has been set, cert-manager will set the default
requested usages of `digital signature`, `key encipherment`, and `server auth`.
cert-manager will not attempt to request a new certificate if the current
certificate does not match the current key usage set.

An exhaustive list of supported key usages can be found in the [API reference
documentation](../reference/api-docs.md#cert-manager.io/v1.KeyUsage).

<h2 id="temporary-certificates-whilst-issuing">Temporary Certificates while Issuing</h2>

On old GKE versions (`1.10.7-gke.1` and below), when requesting certificates
[using the ingress-shim](./ingress.md) alongside the
[`ingress-gce`](https://cloud.google.com/kubernetes-engine/docs/concepts/ingress)
ingress controller, `ingress-gce`
[required](https://github.com/kubernetes/ingress-gce/pull/388) a temporary
certificate must be present while waiting for the issuance of a signed
certificate. Note that this issue was
[solved](https://github.com/cert-manager/cert-manager/issues/606#issuecomment-424397233)
in `1.10.7-gke.2`.

```yaml
# Required for GKE 1.10.7-gke.1 and below.
cert-manager.io/issue-temporary-certificate": "true"
```

That made sure that a temporary self-signed certificate was present in the
`Secret`. The self-signed certificate was replaced with the signed certificate
later on.

<h2 id="rotation-private-key">Rotation of the private key</h2>

By default, the private key won't be rotated automatically. Using the setting
`rotationPolicy: Always`, the private key Secret associated with a Certificate
object can be configured to be rotated as soon as an action triggers the
reissuance of the Certificate object (see
[Actions that will trigger a rotation of the private key](#actions-triggering-private-key-rotation) below).

With `rotationPolicy: Always`, cert-manager waits until the Certificate
object is correctly signed before overwriting the `tls.key` file in the
Secret.

With this setting, you can expect **no downtime** if your application can detect
changes to the mounted `tls.crt` and `tls.key` and reload them gracefully or
automatically restart.

If your application only loads the private key and signed certificate once
at start up, the new certificate won't immediately be served by your
application, and you will want to either manually restart your pod with
`kubectl rollout restart`, or automate the action by running
[wave](https://github.com/wave-k8s/wave). Wave is a Secret controller that
makes sure deployments get restarted whenever a mounted Secret changes.

!!! alert

    Re-use of private keys

    Some issuers, like the built-in [Venafi
    issuer](../configuration/venafi.md), may disallow re-using private keys.
    If this is the case, you must explicitly configure the `rotationPolicy:
    Always` setting for each of your Certificate objects accordingly.

In the following example, the certificate has been set with
`rotationPolicy: Always`:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
spec:
  secretName: my-cert-tls
  privateKey:
    rotationPolicy: Always # 🔰 Here.
```

<h3 id="actions-triggering-private-key-rotation">Actions that will trigger a rotation of the private key</h3>

Setting the `rotationPolicy: Always` won't rotate the private key immediately.
In order to rotate the private key, the certificate objects must be reissued. A
certificate object is reissued under the following circumstances:

- when the X.509 certificate is nearing expiry, which is when the Certificate's
  `status.renewalTime` is reached;
- when a change is made to one of the following fields on the Certificate's
  spec: `commonName`, `dnsNames`, `ipAddresses`, `uris`, `emailAddresses`,
  `subject`, `isCA`, `usages`, `duration` or `issuerRef`;
- when a reissuance is manually triggered with the following:
  ```sh
  cmctl renew cert-1
  ```
  Note that the above command requires [cmctl](../reference/cmctl.md#renew).

!!! "warning"

    **❌** Deleting the Secret resource associated with a Certificate resource is
    **not a recommended solution** for manually rotating the private key. The
    recommended way to manually rotate the private key is to trigger the reissuance
    of the Certificate resource with the following command (requires
    [`cmctl`](../reference/cmctl.md#renew)):

    ```sh
    cmctl renew cert-1
    ```

### `rotationPolicy` 设置

The possible values for `rotationPolicy` are:

| Value                  | Description                                                   |
| ---------------------- | ------------------------------------------------------------- |
| `Never` (default)      | cert-manager reuses the existing private key on each issuance |
| `Always` (recommended) | cert-manager regenerates a new private key on each issuance   |

With `rotationPolicy: Never`, a private key is only generated if one does not
already exist in the target Secret resource (using the `tls.key` key). All
further issuances will re-use this private key. This is the default in order to
maintain compatibility with previous releases.

With `rotationPolicy: Always`, a new private key will be generated each time an
action triggers the reissuance of the certificate object (see [Actions that will
trigger a rotation of the private key](#actions-triggering-private-key-rotation)
above). Note that if the private key secret already exists when creating the
certificate object, the existing private key will not be used, since the
rotation mechanism also includes the initial issuance.

!!! info

    👉 We recommend that you configure `rotationPolicy: Always` on your Certificate
    resources. Rotating both the certificate and the private key simultaneously
    prevents the risk of issuing a certificate with an exposed private key. Another
    benefit to renewing the private key regularly is to let you be confident that
    the private key rotation can be done in case of emergency. More generally, it is
    a good practice to be rotating the keys as often as possible, reducing the risk
    associated with compromised keys.

## 清除证书删除时的秘密

By default, cert-manager does not delete the `Secret` resource containing the signed certificate when the corresponding `Certificate` resource is deleted.
This means that deleting a `Certificate` won't take down any services that are currently relying on that certificate, but the certificate will no longer be renewed.
The `Secret` needs to be manually deleted if it is no longer needed.

If you would prefer the `Secret` to be deleted automatically when the `Certificate` is deleted, you need to configure your installation to pass the `--enable-certificate-owner-ref` flag to the controller.

## 更新

cert-manager will automatically renew `Certificate`s. It will calculate _when_ to renew a `Certificate` based on the issued X.509 certificate's duration and a 'renewBefore' value which specifies _how long_ before expiry a certificate should be renewed.

`spec.duration` and `spec.renewBefore` fields on a `Certificate` can be used to specify an X.509 certificate's duration and a 'renewBefore' value. Default value for `spec.duration` is 90 days. Some issuers might be configured to only issue certificates with a set duration, so the actual duration may be different.
Minimum value for `spec.duration` is 1 hour and minimum value for `spec.renewBefore` is 5 minutes.
It is also required that `spec.duration` > `spec.renewBefore`.

Once an X.509 certificate has been issued, cert-manager will calculate the renewal time for the `Certificate`. By default this will be 2/3 through the X.509 certificate's duration. If `spec.renewBefore` has been set, it will be `spec.renewBefore` amount of time before expiry. cert-manager will set `Certificate`'s `status.RenewalTime` to the time when the renewal will be attempted.

## 其他证书输出格式

!!! warning

    ⛔️ The additional certificate output formats feature is currently in an
    _experimental_ alpha state, and is subject to breaking changes or complete
    removal in future releases. This feature is only enabled by adding it to the
    `--feature-gates` flag on the cert-manager controller and webhook components:

    ```bash
    --feature-gates=AdditionalCertificateOutputFormats=true
    ```

`additionalOutputFormats` is a field on the Certificate `spec` that allows
specifying additional supplementary formats of issued certificates and their
private key. There are currently two supported additional output formats:
`CombinedPEM` and `DER`. Both output formats can be specified on the same
Certificate.

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
spec:
  ...
  secretName: my-cert-tls
  additionalOutputFormats:
  - type: CombinedPEM
  - type: DER

# Results in:

apiVersion: v1
kind: Secret
metadata:
  name: my-cert-tls
type: kubernetes.io/tls
data:
  ca.crt: <PEM CA certificate>
  tls.key: <PEM private key>
  tls.crt: <PEM signed certificate chain>
  tls-combined.pem: <PEM private key + "\n" + PEM signed certificate chain>
  key.der: <DER binary format of private key>
```

#### `CombinedPEM`

The `CombinedPEM` type will create a new key entry in the resulting
Certificate's Secret `tls-combined.pem`. This entry will contain the PEM encoded
private key, followed by at least one new line character, followed by the PEM
encoded signed certificate chain-

```text
<private key> + "\n" + <signed certificate chain>
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-cert-tls
type: kubernetes.io/tls
data:
  tls-combined.pem: <PEM private key + "\n" + PEM signed certificate chain>
  ...
```

#### `DER`

The `DER` type will create a new key entry in the resulting Certificate's Secret
`key.der`. This entry will contain the DER binary format of the private key.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-cert-tls
type: kubernetes.io/tls
data:
  key.der: <DER binary format of private key>
  ...
```
