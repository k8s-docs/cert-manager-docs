---
title: "兼容性"
linkTitle: ""
weight: 100
type: "docs"
---

下面，你会发现各种兼容性问题和怪现象细节 你可以通过在您的环境影响.

## GKE

当谷歌配置控制平面的私人集群, 他们会自动配置 VPC 你 Kubernetes 群集的网络和一个单独的谷歌管理项目间的对等.

为了限制什么谷歌能够在群集内的访问, 防火墙规则配置限制访问您 Kubernetes pods.
这将意味着你将体验到`webhook`不工作和经验的错误如 `Internal error occurred: failed calling admission webhook ... the server is currently unable to handle the request`.

为了使用网络挂接组件与 GKE 专用群集, 您必须配置额外的防火墙规则，以允许您的 webhook pod GKE 控制平面的访问.

你可以在[GKE 文档](https://cloud.google.com/kubernetes-engine/docs/how-to/private-clusters#add_firewall_rules)阅读更多关于如何添加防火墙规则的 GKE 控制平面节点.

## Webhook

自从 `v0.14` 禁用 webhook 不支持了 .
