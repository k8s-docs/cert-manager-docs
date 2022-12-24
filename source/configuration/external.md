# 外部

cert-manager 支持外部`Issuer`类型。
虽然外部颁发者没有在主证书管理器存储库中实现，但它们在其他方面与任何其他颁发者相同。

外部发行者通常部署为一个 pod，它被配置为监视集群中`CertificateRequest`源，这些源的`issuerRef`与发行者的名称匹配。
外部发行者存在于`cert-manager.io`组之外。

每个发行者的安装可能不同;请查看每个外部发行者的文档，以获得有关安装、配置和使用它的更多详细信息。

## 已知的外部发行人

如果您已经创建了一个想要共享的外部发行者，[提出一个 Pull Request](https://github.com/cert-manager/website/pulls)将它添加到这里!

众所周知，这些外部发行人支持并尊重[批准](https://cert-manager.io/docs/concepts/certificaterequest/#approval).

- [kms-issuer](https://github.com/Skyscanner/kms-issuer): 请求使用[AWS KMS](https://aws.amazon.com/kms/)非对称密钥签名的证书。
- [aws-privateca-issuer](https://github.com/cert-manager/aws-privateca-issuer): Requests
  certificates from [AWS Private Certificate Authority](https://aws.amazon.com/certificate-manager/private-certificate-authority/)
  for cloud native/hybrid environments.
- [google-cas-issuer](https://github.com/jetstack/google-cas-issuer): Used
  to request certificates signed by private CAs managed by the
  [Google Cloud Certificate Authority Service](https://cloud.google.com/certificate-authority-service/).
- [origin-ca-issuer](https://github.com/cloudflare/origin-ca-issuer): Used
  to request certificates signed by
  [Cloudflare Origin CA](https://developers.cloudflare.com/ssl/origin-configuration/origin-ca)
  to enable TLS between Cloudflare edge and your Kubernetes workloads.
- [step-issuer](https://github.com/smallstep/step-issuer): Requests
  certificates from the [Smallstep](https://smallstep.com) [Certificate Authority server](https://github.com/smallstep/certificates).
- [freeipa-issuer](https://github.com/guilhem/freeipa-issuer): Requests
  certificates signed by [FreeIPA](https://www.freeipa.org).
- [ADCS Issuer](https://github.com/nokia/adcs-issuer): Requests
  certificates signed by [Microsoft Active Directory Certificate Service](https://docs.microsoft.com/en-us/windows-server/networking/core-network-guide/cncg/server-certs/install-the-certification-authority).
  [NOT MAINTAINED]
- [CFSSL Issuer](https://gerrit.wikimedia.org/r/plugins/gitiles/operations/software/cfssl-issuer/): Request certificates signed by a [CFSSL](https://github.com/cloudflare/cfssl) `multirootca` instance.
- [ncm-issuer](https://github.com/nokia/ncm-issuer): Requests certificates from the [Nokia](https://www.nokia.com/) [Netguard Certificate Manager](https://www.nokia.com/networks/security-portfolio/netguard/certificate-manager)
- [tcs-issuer](https://github.com/intel/trusted-certificate-issuer) Requests certificates signed securely using [Intel's SGX technology](https://www.intel.com/content/www/us/en/developer/tools/software-guard-extensions/overview.html).

## 建立新的外部发行人

如果您对构建一个新的外部发行方感兴趣，请查看[开发文档](../contributing/external-issuers.md).
