---
title: "ACME Orders 和 Challenges"
linkTitle: "ACME"
weight: 350
type: "docs"
---

cert-manager 支持请求从 ACME 服务器证书, 包括 [Let's Encrypt](https://letsencrypt.org/), 随着使用 [ACME Issuer](../../configuration/acme/).
这些证书是由大多数计算机通常信任的公共互联网上.
为了成功地申请一个证书, `cert-manager` 必须解决 ACME 挑战 这是为了证明该客户拥有正在请求的 DNS 地址完成.

为了完成这些挑战, `cert-manager` 推出两款 `CustomResource` 类型; `Orders` 和 `Challenges`.

## Orders

`Order`资源使用的 ACME 发行来管理 ACME 的生命周期`order` 已签署 TLS 证书.
在 ACME 订单和域验证的更多细节可以在 Let's Encrypt 网站[这里](https://letsencrypt.org/how-it-works/)找到.
订单表示单个证书请求 它会自动一旦新创建[`CertificateRequest`](../certificaterequest/) 资源引用一个 ACME 发行人已建立. `CertificateRequest` 资源是通过证书管理器自动创建一旦 [`Certificate`](../certificate/) 创建资源, 有其规格改变, 或者需要更新.

作为最终用户, 你不需要手动创建一个`Order`资源.
一旦创建，一个`Order`不能改变.
相反，必须创建一个新的`Order`资源.

该`Order`资源封装多个 ACME`challenges`为`order`, 正因为如此, 将管理一个或多个`Challenge`资源.

## Challenges

`Challenge`资源用于由 ACME 发行者来管理必须以完成对单个 DNS 名称/标识的 `authorization` 内完成的 ACME `challenge`的生命周期.

当创建一个 `Order` 资源, 顺序控制器将创建为正在授权与 ACME 服务器的每个 DNS 名称 `Challenge` 资源.

作为最终用户, 你不需要手动创建一个 `Challenge` 资源.
一旦创建, 一个 `Challenge` 不能改变.
相反，必须创建一个新的 `Challenge` 资源.

### Challenge 生命周期

一个`Challenge`资源被创建后，它最初将排队等待处理.
处理才会开始面对的挑战是`scheduled`启动.
正在尝试在这一次调度过程中防止太多的挑战, 或相同的 DNS 名称的多重挑战正在尝试一次.
有关挑战是如何安排的详细信息, 读出的[挑战调度](./#challenge-scheduling).

一旦挑战已定, 以确定其当前状态时，它会先`synced`与 ACME 服务器.
如果挑战已经是有效的, 它`state`将更新为`valid`, 并且还将设置 `status.processing = false` 至 'unschedule' 本身.

如果挑战仍然是`pending`, 使用配置的解算器的挑战控制器将`present`挑战, HTTP01 或 DNS01 之一.
一旦挑战已经`presented`, 它将设置 `status.presented = true`.

一旦 'presented', 挑战控制器将执行 'self check' 以确保挑战有 'propagated' (即权威 DNS 服务器已经更新到正确响应, 或更改到入口资源已经由入口控制器观察和使用).

如果自检失败, `cert-manager` 将重试自检具有固定 10 秒重试间隔.
Challenges 不过完成自我检查将继续通过任一重试`Order`重试直到用户介入(通过删除`Order`资源)或修改相关的`Certificate`资源解决任何配置错误.

一旦自检查合格, 在 ACME 'authorization' 关联的 这个挑战将是 'accepted'.

授权的最终状态在接受之后它会被复制对面的挑战 `status.state` 字段, 还有 'error reason' 如果发生错误而 ACME 服务器尝试验证的挑战.

一旦挑战赛已经进入了 `valid`, `invalid`, `expired` or `revoked` 状态, 它将设置 `status.processing = false` 为了防止 ACME 挑战的任何进一步的处理, 并允许调度的另一个挑战 如果有，完成挑战积压.

### Challenge 调度

而是试图处理所有挑战一次的, 挑战是通过证书管理器`scheduled`.

该调度适用于同时挑战以及不允许对同一 DNS 名称两个方面的挑战和解决者(`HTTP01` 要么 `DNS01`)的最大数量上限键入要在一次完成.

的，可以在一个时间被处理的挑战的最大数目是 60 作为[`ddff78`](https://github.com/jetstack/cert-manager/blob/ddff78f011558e64186d61f7c693edced1496afa/pkg/controller/acmechallenges/scheduler/scheduler.go#L31-L33).
