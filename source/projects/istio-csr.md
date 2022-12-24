---
title: istio-csr
description: ""
---

# istio-csr

istio-csr 是一个代理，允许使用[cert-manager](https://cert-manager.io)保护[Istio](https://istio.io)工作负载和控制平面组件。

促进 mTLS 的证书(集群间和集群内)将通过[证书管理器颁发者](https://cert-manager.io/docs/concepts/issuer)签署、交付和更新。

## istio-csr 入门指南

We have [a guide](../tutorials/istio-csr/istio-csr.md) for setting up istio-csr in a fresh
[kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) cluster.

Following the guide is the best way to see istio-csr in action.

If you've already seen istio-csr in action or if you're experienced with running
Istio and just want quick installation instructions, read on for more details.

## 底层细节(适用于有经验的 Istio 用户)

⚠️ The [getting started](../tutorials/istio-csr/istio-csr.md) guide is a better place if you just want to try istio-csr out!

Running istio-csr requires a few steps and preconditions in order:

1. A cluster _without_ Istio already installed
2. cert-manager [installed](https://cert-manager.io/docs/installation/) in the cluster
3. An `Issuer` or `ClusterIssuer` which will be used to issue Istio certificates
4. istio-csr installed (likely via helm)
5. Istio [installed](https://istio.io/latest/docs/setup/install/istioctl/) with
   some custom config required, e.g. using the example config from the [repository](https://github.com/cert-manager/istio-csr/tree/main/hack).

### 为什么自定义 Istio 安装清单?

If you take a look at the contents of [the example Istio install
manifests](https://github.com/cert-manager/istio-csr/tree/main/hack)
there are a few custom configuration options which are important.

Required changes include setting `ENABLE_CA_SERVER` to `false` and setting the `caAddress` from which Istio will
request certificates; replacing the CA server is the whole point of istio-csr!

Mounting and statically specifying the root CA is also an important recommended step. Without a manually specified
root CA istio-csr defaults to trying to discover root CAs automatically, which could theoretically lead to a
[signer hijacking attack](https://github.com/cert-manager/istio-csr/issues/103#issuecomment-923882792) if for example
a signer's token was stolen (such as the cert-manager controller's token).

### Issuer 还是 ClusterIssuer?

Unless you know you need a `ClusterIssuer` we'd recommend starting with an `Issuer`, since it should be easier to reason about
the access controls for an Issuer; they're namespaced and so naturally a little more limited in scope.

That said, if you view your entire Kubernetes cluster as being a trust domain itself, then a ClusterIssuer is the more natural
fit. The best choice will depend on your specific situation.

Our [getting started guide](../tutorials/istio-csr/istio-csr.md) uses an `Issuer`.

### 哪种发行者类型?

Whether you choose to use an `Issuer` or a `ClusterIssuer`, you'll also need to choose the type of issuer you want such as:

- [CA](https://cert-manager.io/docs/configuration/ca/)
- [Vault](https://cert-manager.io/docs/configuration/vault/)
- or an [external issuer](https://cert-manager.io/docs/configuration/external/)

The key requirement is that arbitrary values can be placed into the `subjectAltName` (SAN) X.509 extension, since
Istio places SPIFFE IDs there.

That means that the ACME issuer **will not work** &mdash; publicly trusted certificates such as those issued by Let's Encrypt
don't allow arbitrary entries in the SAN, for very good reasons.

If you're already using [HashiCorp Vault](https://www.vaultproject.io/) then the Vault issuer is an obvious choice. If
you want to control your own PKI entirely, we'd recommend the CA issuer. The choice is ultimately yours.

### 在 Istio 之后安装 istio-csr

This is unsupported because it's exceptionally difficult to do safely. It's likely that installing istio-csr _after_ Istio isn't
possible to do without downtime, since installing istio-csr second would require a time period where all Istio sidecars trust
both the old Istio-managed CA and the new cert-manager controlled CA.

## istio-csr 如何工作?

istio-csr implements the gRPC Istio certificate service which authenticates,
authorizes, and signs incoming certificate signing requests from Istio
workloads, routing all certificate handling through cert-manager installed in
the cluster.

This seamlessly matches the behavior of istiod in a typical installation, while
allowing certificate management through cert-manager.
