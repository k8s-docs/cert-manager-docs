---
title: 诊断 ACME / Let's Encrypt Certificates 问题
description: 了解如何在证书管理器无法更新ACME / Let's Encrypt证书时诊断问题。
---

# ACME / Let's Encrypt 证书问题诊断

了解如何在证书管理器无法更新 ACME / Let's Encrypt 证书时诊断问题。

## 概述

当请求 ACME 证书时，证书管理器将创建`Order` 和 `Challenges`来完成请求。因此，如果过程中出现问题，可以使用更多的资源进行调查和调试。
您可以在[概念页](../concepts/acme-orders-challenges.md)中阅读有关这些资源的更多信息。

在开始之前，您可能应该看一下我们的[通用故障排除指南](./README.md)

## 1. 故障诊断(集群)颁发者

首先，检查您正在使用的(集群)颁发者是否处于就绪状态:

```bash
$ kubectl get issuer
$ kubectl get clusterissuer
NAME               READY   AGE
letsencrypt        True    38m
letsencrypt-http   False    32m
```

如果你看到`False`，使用`kubectl describe`检查状态。例如:

```bash
$ kubectl describe issuer letsencrypt-http
$ kubectl describe clusterissuer letsencrypt-http
Name:         letsencrypt
API Version:  cert-manager.io/v1
Kind:         Issuer
Spec:
  Acme:
    Email:            cert-manager@example.com
    Private Key Secret Ref:
      Name:  letsencrypt
    Server:  https://acme-staging-v02.api.letsencrypt.org/directory
Status:
  Acme:
  Conditions:
    Message:               Failed to update ACME account:400 urn:ietf:params:acme:error:invalidEmail: Unable to update account :: invalid contact domain. Contact emails @example.com are forbidden
    Reason:                ErrUpdateACMEAccount
    Status:                False
    Type:                  Ready
Events:
  Type     Reason                Age                  From          Message
  ----     ------                ----                 ----          -------
  Warning  ErrUpdateACMEAccount  101s (x3 over 106s)  cert-manager  Failed to update ACME account:400 urn:ietf:params:acme:error:invalidEmail: Unable to update account :: invalid contact domain. Contact emails @example.com are forbidden
```

### 常见的错误

!!! Failure "Failed to update ACME account:400 urn:ietf:params:acme:error:invalidEmail"

    您在`Issuer`配置中指定的电子邮件无效。

!!! Failure "Error initializing issuer: Failed to register ACME account: secrets 'acme-key' already exists"

    可能有来自前一个`Issuer`的剩余帐户不再有效，您应该删除`secret`，以便重新创建它。

!!! Failure "Error accepting challenge: 400 urn:ietf:params:acme:error:malformed: Unable to update challenge :: authorization must be pending"

    这表明当证书管理器向ACME服务器发送请求以接受质询时，授权并没有处于'pending' 状态。
    这可能是因为域验证已经失败，授权被标记为'invalid'。
    在`Order` 或 `Challenge` 的状态上检查授权URL，以查看授权的状态和任何其他信息。

## 2. 故障诊断 Orders

当我们在`CertificateRequest`资源上运行描述时，我们看到一个`Order`已经被创建:

```bash
$ kubectl describe certificaterequest example-com-2745722290
...
Events:
  Type    Reason        Age   From          Message
  ----    ------        ----  ----          -------
  Normal  OrderCreated  5s    cert-manager  Created Order resource default/example-com-2745722290-439160286
```

订单是向 ACME 实例发出证书的请求。
通过在特定的顺序上运行`kubectl describe order`，可以收集到进程中失败的信息:

```console
$ kubectl describe order example-com-2745722290-439160286
...
Reason:
State:         pending
URL:           https://acme-v02.api.letsencrypt.org/acme/order/41123272/265506123
Events:
  Type    Reason   Age   From          Message
  ----    ------   ----  ----          -------
  Normal  Created  1m    cert-manager  Created Challenge resource "example-com-2745722290-439160286-0" for domain "test1.example.com"
  Normal  Created  1m    cert-manager  Created Challenge resource "example-com-2745722290-439160286-1" for domain "test2.example.com"
```

在这里，我们可以看到证书管理器创建了两个 Challenge 资源来验证我们控制特定的域，这是 ACME 获取已签名证书的要求。

然后，您可以继续运行`kubectl describe challenge example-com-2745722290-439160286-0`来进一步调试订单的进度。

一旦订单成功，您应该看到如下所示的事件:

```bash
$ kubectl describe order example-com-2745722290-439160286
...
Reason:
State:         valid
URL:           https://acme-v02.api.letsencrypt.org/acme/order/41123272/265506123
Events:
  Type    Reason      Age   From          Message
  ----    ------      ----  ----          -------
  Normal  Created     72s   cert-manager  Created Challenge resource "example-com-2745722290-439160286-0" for domain "test1.example.com"
  Normal  Created     72s   cert-manager  Created Challenge resource "example-com-2745722290-439160286-1" for domain "test2.example.com"
  Normal  OrderValid  4s    cert-manager  Order completed successfully
```

您可以从`Order`的状态中看到有关[ACME 授权](https://datatracker.ietf.org/doc/html/rfc8555#section-7.1.4)状态的一些附加信息，需要使用授权 URL 作为此订单的一部分进行验证:

```bash
$ kubectl get order <order-name> -ojsonpath='{.status.authorizations[x].url}'
```

如果订单没有成功完成，您可以通过在`Challenge`资源上运行`kubectl describe`来调试订单的挑战，该资源将在以下步骤中描述。

## 3. 故障诊断 Challenges

为了确定 ACME 订单没有完成的原因，我们可以使用 cert-manager 创建的`Challenge`资源进行调试。

为了确定哪个 `Challenge`失败了，你可以运行`kubectl get challenges`:

```console
$ kubectl get challenges
...
NAME                                 STATE     DOMAIN            REASON                                     AGE
example-com-2745722290-4391602865-0  pending   example.com       Waiting for dns-01 challenge propagation   22s
```

这表明挑战已经使用 DNS01 求解器成功提出，现在 cert-manager 正在等待'self check'通过。

你可以通过使用`kubectl describe`来获得更多关于挑战及其生命周期的信息:

```bash
$ kubectl describe challenge example-com-2745722290-4391602865-0
...
Status:
  Presented:   true
  Processing:  true
  Reason:      Waiting for dns-01 challenge propagation
  State:       pending
Events:
  Type    Reason     Age   From          Message
  ----    ------     ----  ----          -------
  Normal  Started    19s   cert-manager  Challenge scheduled for processing
  Normal  Presented  16s   cert-manager  Presented challenge using dns-01 challenge mechanism
```

关于每个挑战状态的进展将被记录为事件或在挑战的`status`块上(如上图所示)。

在 DNS01 的情况下，你会发现任何错误从你的 DNS 提供商在这里。

在证书管理器向 ACME 提供程序提出挑战之前，HTTP01 和 DNS01 都要先进行"self-check"。
这样做不是为了让由于 DNS 或负载均衡器传播而导致挑战失败而使 ACME 提供程序过载。
这的状态可以在 `description` 的 `status` 块中找到:

```console
$ kubectl describe challenge
[...]
Status:
  Presented:   true
  Processing:  true
  Reason:      Waiting for http-01 challenge propagation: failed to perform self check GET request 'http://example.com/.well-known/acme-challenge/_fgdLz0i3TFiZW4LBjuhjgd5nTOkaMBhxYmTY': Get "http://example.com/.well-known/acme-challenge/_fgdLz0i3TFiZW4LBjuhjgd5nTOkaMBhxYmTY: remote error: tls: handshake failure
  State:       pending
[...]
```

在本例中，由于网络问题，HTTP01 检查失败。
在这里，您还将看到来自 DNS 提供商的任何错误。

您还可以从`Challenge`的状态中看到关于挑战应该使用授权 URL 验证的[ACME 授权](https://datatracker.ietf.org/doc/html/rfc8555#section-7.1.4)状态的一些附加信息:

```bash
$ kubectl get challenge <challenge-name> -ojsonpath='{.spec.authorizationURL}'
```

### HTTP01 故障排除

首先检查您是否可以从公共互联网上看到挑战 URL，如果不能正常工作，请检查您的入口和防火墙配置以及为解决 ACME 挑战而创建的服务和 pod cert-manager。
如果这可以工作，检查您的集群是否也可以看到它。从 Pod 内部进行测试是很重要的。如果你得到一个连接错误，建议检查集群的网络配置。
如果你收到一个`tls: handshake failure`，请尝试在 Ingress 或 Certificate 资源上设置注释`cert-manager.io/issue-temporary-certificate: "true"`。这将在颁发实际证书之前为入口控制器颁发一个临时的自签名证书。
如果您仍然有问题，可能是您的入口控制器处理相同主机名的多个资源时出现了问题，在这种情况下，可能需要注释`acme.cert-manager.io/http01-edit-in-place: "true"`。

例如，当使用 GKE 和谷歌云负载均衡器时，建议设置:

```
cert-manager.io/issue-temporary-certificate: "true"
acme.cert-manager.io/http01-edit-in-place: "true"
```

这将允许谷歌云负载均衡器使用临时证书正确传播 HTTPS 端点，`http01-edit-in-place`部分将阻止 GKE 为挑战端点分配第二个 IP 地址。

#### 404 状态码

如果您的挑战自检失败，出现 404 not found 错误。请务必检查以下内容:

- 你可以从公共互联网上访问 URL
- ACME 求解器 Pod 已经启动并运行
- 使用`kubectl describe ingress`来检查 HTTP01 求解器入口的状态。(除非您使用`acme.cert-manager.io/http01-edit-in-place`，然后检查与您的域相同的入口)

### DNS01 故障排除

如果您没有看到关于您的 DNS 提供商的错误事件，您可以检查以下检查您是否可以从公共互联网或您的 DNS 提供商的界面中看到`_acme_challenge.domain` TXT DNS 记录。
cert-manager 将通过查询集群的 DNS 解决程序来检查 DNS 记录是否已经传播。
如果你能从公共互联网上看到它，但不能从集群内部看到它，你可能想要改变[自我检查的 DNS 服务器](../configuration/acme/dns01/README.md#setting-nameservers-for-dns01-self-check)，因为一些云提供商会在内部覆盖 DNS。

#### 证书管理器为您的域名识别了错误的区域

默认情况下，cert-manager 使用 SOA(授权开始)记录来确定在您的 DNS 提供商使用哪个区域名称。
一些 DNS 解析器将过滤此信息，如果在这种情况下 cert-manager 无法确定区域，建议[为 DNS01 自检更改 DNS 服务器](../configuration/acme/dns01/README.md#setting-nameservers-for-dns01-self-check)。

如果您使用`dnsmasq`作为您的 DNS 服务器，如果您使用[`--filterwin2k`标志][thekelleys]，可能会发生这种情况。
在[OpenWRT 中有一个`filterwin2k`配置选项][openwrt]。
在[LuCI 中有一个"Filter useless"选项][dhcp]。
通过启用这个标志，`dnsmasq`将删除所有`SOA`记录。

[thekelleys]: http://www.thekelleys.org.uk/dnsmasq/docs/setup.html
[openwrt]: https://openwrt.org/docs/guide-user/base-system/dhcp#all_options
[dhcp]: (https://github.com/openwrt/luci/blob/15757dd5b18f9e00ba3c9b38af4d46702a31fe33/modules/luci-mod-network/htdocs/luci-static/resources/view/network/dhcp.js#L217-L219)

## 2020 年 3 月 Let's Encrypt CAA 复查漏洞

在[3 月 4 日的公告](https://community.letsencrypt.org/t/revoking-certain-certificates-on-march-4/114864)Let's Encrypt 将撤销一些证书，因为他们验证 CAA 记录的方式存在漏洞，我们创建了一个工具来分析您现有的证书管理器管理的证书，并将其序列号与已公布的已撤销证书列表进行比较。
建议 Let's Encrypt & cert-manager 的所有用户使用此工具进行检查，以确保他们在集群中没有遇到任何无效的证书错误。
你可以在这里找到检查工具的副本: <https://github.com/jetstack/letsencrypt-caa-bug-checker>。
