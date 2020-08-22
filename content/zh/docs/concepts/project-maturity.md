---
title: "项目成熟度(Project Maturity)"
linkTitle: "项目成熟度"
weight: 0
type: "docs"
---

cert-manager 具有广泛采用的 Kubernetes 社区 同它使用的是在生产和非生产集群.
该项目仍处于 α 状态 它尚未达到`v1.0`.

## API

该证书管理器 API 当前处于`v1alpha2`版本，因此可随时更改。
我们预计`v1beta1`的一个测试版本 在此，我们希望最小的变化, 如果有的话, 为`v1`的下一个版本发布.
我们预计到打版`v1` 2019 末, 早 2020.

## 兼容性

cert-manager 拥有兼容当前稳定的上游 Kubernetes 版本硬保障.
超出此, `cert-manager` 的目的还在于在兼容的版本到`N-4`的, 其中`N`是当前版本上游释放.
这意味着，如果当前版本 `v0.16`, `cert-manager` 旨在与版本降到`v0.12`兼容.
这是通过对 Kubernetes 的每个版本运行周期结束到终端的测试工作完成.

版本比当前版本 Kubernetes 下降到`N-4`的不能保证降低.
虽然考虑将作出保证兼容尽可能多的版本尽可能, 有时需要失去兼容性进一步的`cert-manager`的功能集，使较新的使用的兴趣在上游 Kubernetes 可用功能.

因为`cert-manager`版本`v0.11`的, 支持的最低 Kubernetes 版本是`v1.12`.
