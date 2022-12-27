---
title: Frequently Asked Questions (FAQ)
description: Find answers to some frequently asked questions about cert-manager
---

# 常见问题 (FAQ)

在本页上，您将找到有关 cert-manager 的一些常见问题的答案。

## 术语

??? question "`publicly trusted` 和 `self-signed`是什么意思?"

    这些术语的定义见[TLS术语页](../reference/tls-terminology.md).

??? question "“根证书”、“中间证书”和“叶证书”是什么意思?"

    These terms are defined in the [TLS Terminology page](../reference/tls-terminology.md).

## 证书

??? question "我可以随意从证书管理器触发续签吗?"

    This is a feature in cert-manager starting in `v0.16` using the `cmctl` CLI. More information can be found on [the renew command's page](../reference/cmctl.md#renew)

??? question "证书什么时候重新颁发?"

    为了确定是否需要重新颁发证书，证书管理器会查看`Certificate`源规范和最新的`CertificateRequest`规范，
    以及`Secret`中包含X.509证书的数据。

    如果出现以下情况，发行过程将始终被触发:

    - 在`Certificate`规范上命名的`Secret`，不存在，缺少私钥或证书数据或包含损坏的数据
    - 存储在`Secret` 中的私钥与`Certificate`上的私钥规格不匹配
    - 已发出证书的公开密匙与储存在`Secret`内的私钥不相符。
    - `Secret`上的cert-manager颁发者注释与`Certificate`上指定的颁发者不匹配
    - 颁发证书上的DNS名称、IP地址、url或电子邮件地址与`Certificate`规范上的不匹配
    - 证书需要更新(因为已过期或更新时间已过)
    - 证书已被标记为手动更新[使用`cmctl`](../reference/cmctl.md#renew)

    此外，如果找到了`Certificate`的最新`CertificateRequest`，在以下情况下，证书管理器也会重新颁发:

    - 在`CertificateRequest`中发现的CSR上的通用名称与`Certificate`规范上的不匹配
    - 在`CertificateRequest`中发现的CSR的主题字段与`Certificate`规范的主题字段不匹配
    - `CertificateRequest`上的持续时间与`Certificate`规范上的持续时间不匹配
    - `Certificate`规范上的 `isCA` 字段值与`CertificateRequest`不匹配
    - `CertificateRequest`规范中的DNS名称、IP地址、url或电子邮件地址与`Certificate`规范中的不匹配
    - `CertificateRequest`规范上的关键用法与`Certificate`规范上的不匹配

    !!! note

        请注意，对于某些字段，只有当存在`CertificateRequest`时才会触发更改后的重新发布，
        证书管理器可以使用`CertificateRequest`来确定`Certificate`的规范自上次发布以来是否发生了更改。
        这是因为某些颁发者可能不尊重这些字段的请求值，因此我们不能依赖已颁发的X.509证书中的值。
        一个这样的字段是`.spec.duration`——如果有一个`CertificateRequest`进行比较，对该字段的更改只会触发重新发布。
        如果您需要重新签发，但由于没有`CertificateRequest`(即在备份和恢复之后)而无法自动触发重新签发，
        您可以使用[`cmctl renew`](../reference/cmctl.md#renew)手动触发它。

??? question "为什么我的根证书不在我颁发的 Secret `tls.crt`中?"

    Occasionally, people work with systems which have made a flawed choice regarding TLS chains. The [TLS spec](https://datatracker.ietf.org/doc/html/rfc5246#section-7.4.2)
    has the following section for the "Server Certificate" section of the TLS handshake:

    > This is a sequence (chain) of certificates. The sender's
    > certificate MUST come first in the list. Each following
    > certificate MUST directly certify the one preceding it. Because
    > certificate validation requires that root keys be distributed
    > independently, the self-signed certificate that specifies the root
    > certificate authority MAY be omitted from the chain, under the
    > assumption that the remote end must already possess it in order to
    > validate it in any case.

    In a standard, secure and correctly configured TLS environment, adding a root certificate to the chain is
    almost always unnecessary and wasteful.

    There are two ways that a certificate can be trusted:

    - explicitly, by including it in a trust store.
    - through a signature, by following the certificate's chain back up to an explicitly trusted certificate.

    Crucially, root certificates are by definition self-signed and they cannot be validated through a signature.

    As such, if we have a client trying to validate the certificate chain sent by the server, the client must already have the
    root before the connection is started. If the client already has the root, there was no point in it being sent by the server!

    The same logic with not sending root certificates applies for servers trying to validate client certificates;
    the [same justification](https://datatracker.ietf.org/doc/html/rfc5246#section-7.4.6) is given in the TLS RFC.

??? question "如何查看与证书对象相关的所有历史事件?"

    cert-manager publishes all events to the Kubernetes events mechanism, you can get the events for your specific resources using `kubectl describe <resource> <name>`.

    Due to the nature of the Kubernetes event mechanism these will be purged after a while. If you're using a dedicated logging system it might be able or is already also storing Kubernetes events.

??? question "如果发行失败会发生什么?它会被重新审判吗?"

    {/_ This empty link preserves old links to #what-happens-if-a-renewal-doesn't happen?-will-it-be-tried-again-after-some-time?", which matched the old title of this section _/}

    <a id="alternative-certificate-chain" className="hidden-link"></a>

    cert-manager will retry a failed issuance except for a few rare edge cases where manual intervention is needed.

    If an issuance fails because of a temporary error, it will be retried again with a short exponential backoff (currently 5 seconds to 5 minutes). A temporary error is one that does not result in a failed `CertificateRequest`.

    If the issuance fails with an error that resulted in a failed `CertificateRequest`, it will be retried with a longer binary exponential backoff (1 hour to 32 hours) to avoid overwhelming external services.

    You can always trigger immediate renewal using the [`cmctl renew` command](../reference/cmctl.md#renew)

??? question "是否支持 ECC(椭圆曲线加密)?"

    cert-manager supports ECDSA key pairs! You can set your certificate to use ECDSA in the `privateKey` part of your Certificate resource.

    For example:

    ```yaml
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
      name: ecdsa
    spec:
      secretName: ecdsa-cert
      isCA: false
      privateKey:
        algorithm: ECDSA
        size: 256
      dnsNames:
        - ecdsa.example.com
      issuerRef: [...]
    ```

??? question "如果`renewBefore` 或 `duration`没有定义，那么默认值是什么?"

    Default `duration` is [90 days](https://github.com/cert-manager/cert-manager/blob/v1.2.0/pkg/apis/certmanager/v1/const.go#L26). If `renewBefore` has not been set, `Certificate` will be renewed 2/3 through its _actual_ duration.

## 杂项

??? question "Kubernetes 有一个内置的`CertificateSigningRequest`API。为什么不用呢?"

    Kubernetes has a [Certificate Signing Requests API],
    and a [`kubectl certificates` command] which allows you to approve certificate signing requests
    and have them signed by the certificate authority (CA) of the Kubernetes cluster.

    This API and CLI have occasionally been misused to sign certificates for use by non-control-plane Pods but this is a mistake.
    For the security of the Kubernetes cluster, it is important to limit access to the Kubernetes certificate authority,
    and it is important that you do not use that certificate authority to sign certificates which are used outside of the control-plane,
    because such certificates increase the opportunity for attacks on the Kubernetes API server.

    In Kubernetes 1.19 the [Certificate Signing Requests API] has reached V1
    and it can be used more generally by following (or automating) the [Request Signing Process].

    cert-manager currently has some [limited experimental support] for this resource.

    [certificate signing requests api]: https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.19/#certificatesigningrequest-v1-certificates-k8s-io
    [`kubectl certificates` command]: https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#certificate
    [request signing process]: https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#request-signing-process
    [limited experimental support]: ../usage/kube-csr.md

??? question "How to write `cert-manager`"

    cert-manager should always be written in lowercase. Even when it would normally be
    capitalized such as in titles or at the start of sentences. A hyphen should always be
    used between the words, don't replace it with a space and don't remove it.
