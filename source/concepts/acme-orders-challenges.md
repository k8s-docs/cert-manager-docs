---
title: ACME Orders and Challenges
description: "cert-manager core concepts: ACME Orders and Challenges"
---

# ACME 订单和挑战

cert-manager 支持使用[ACME 颁发者](../configuration/acme/README.md)从 ACME 服务器(包括[Let's Encrypt](https://letsencrypt.org/))请求证书。
这些证书通常在公共 Internet 上受到大多数计算机的信任。
为了成功请求证书，证书管理器必须解决 ACME 挑战，以证明客户端拥有被请求的 DNS 地址。

为了完成这些挑战，cert-manager 引入了两种`CustomResource` 类型;`Orders` 和 `Challenges`。

## Orders(订单)

ACME 颁发者使用`Order`源管理已签名 TLS 证书的 ACME 'order'的生命周期。
更多关于 ACME 订单和域验证的详细信息可以在[Let’s Encrypt 网站](https://letsencrypt.org/how-it-works/)上找到。
订单表示一个证书请求，该证书请求将在引用 ACME 颁发者的新[`CertificateRequest`](./certificaterequest.md)源创建之后自动创建。
一旦[`Certificate`](./certificate.md)源被创建，其规格被更改或需要更新，`CertificateRequest`源就会由 cert-manager 自动创建。

作为最终用户，您永远不需要手动创建`Order` 源。
一旦创建，`Order` 就不能更改。
相反，必须创建一个新的`Order` 源。

`Order`源为该'order'封装了多个 ACME'challenges'，因此，将管理一个或多个`Challenge`源。

## Challenge(挑战)

ACME 颁发者使用`Challenge`源来管理必须完成的 ACME'challenge'的生命周期，以完成单个 DNS 名称/标识符的'authorization'。

创建`Order`源时，订单控制器将为 ACME 服务器正在授权的每个 DNS 名称创建`Challenge`源。

作为最终用户，您永远不需要手动创建`Challenge`源。
一旦创建，`Challenge`就不能更改。
相反，必须创建一个新的`Challenge`源。

### 挑战生命周期

在一个`Challenge`源被创建之后，它最初会被排队等待处理。
直到挑战被'scheduled'开始，处理才会开始。
这个调度过程可以防止一次尝试过多的挑战，或者一次尝试同一个 DNS 名称的多个挑战。
有关如何调度挑战的更多信息，请阅读[挑战调度](#challenge-scheduling)。

一旦安排了挑战，它将首先与 ACME 服务器'synced'，以确定其当前状态。
如果挑战已经有效，它的'state'将被更新为 'valid',，并将`status.processing = false`设置为'unschedule'。

如果挑战仍然'pending'，挑战控制器将使用已配置的解决程序(HTTP01 或 DNS01 之一)'present'挑战。
一旦挑战被'presented'，它将设置`status.presented = true`。

一旦'presented'，挑战控制器将执行'self check'，以确保挑战已'propagated'(即，权威 DNS 服务器已更新以正确响应，或入口控制器已观察到入口源的更改并正在使用)。

如果自检失败，cert-manager 将以固定的 10 秒间隔重试自检。
没有完成自检的挑战将继续重试，直到用户通过重试`Order` (删除`Order`源)或修改相关的`Certificate`源来解决任何配置错误。

一旦自检通过，与此挑战相关联的 ACME 'authorization'将被'accepted'.。

接受授权后的最终状态将被复制到挑战的`status.state`字段，以及'error reason' (如果在 ACME 服务器试图验证挑战时发生错误)。

一旦挑战进入`valid`, `invalid`, `expired` 或 `revoked`状态，它将设置`status.processing = false`以防止对 ACME 挑战进行任何进一步的处理，并允许在有积压的挑战需要完成时安排另一个挑战。

### 挑战调度

挑战不是试图一次处理所有挑战，而是由 cert-manager '调度'。

这个调度器对同时挑战的最大数量应用一个上限，并且不允许同一 DNS 名称和求解器类型(`HTTP01` or `DNS01`)的两个挑战同时完成。

到 [`ddff78`](https://github.com/cert-manager/cert-manager/blob/ddff78f011558e64186d61f7c693edced1496afa/pkg/controller/acmechallenges/scheduler/scheduler.go#L31-L33)为止，一次可以处理的最大挑战数是 60 个。.
