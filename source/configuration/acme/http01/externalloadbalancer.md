# 外部负载均衡器

ACME HTTP-01 挑战使用外部负载均衡器

当您使用任何主机提供的外部负载均衡器时，您可能会面临几个配置问题，以使其与 cert-manager 一起工作。

本文档旨在帮助为外部负载均衡器后面的实例配置 HTTP-01 挑战类型。

## NAT Loopback / Hairpin

第一个配置点是 NAT 环回。
由于负载均衡器阻止其后面的实例访问其外部接口，您可能会面临检查问题。

一些网络负载均衡器由于几个原因有这种限制。
它可以通过`iptables`重路由配置，即`NAT loopback`进行配置。

要检查你是否遇到了这个问题:

1. 检查挑战的终点是否对公众开放: `curl <endpoint>`
2. 检查挑战端点不能从负载均衡器后面的内部访问:使用 SSH 在负载均衡器后面的节点上打开一个会话;然后启动与之前相同的命令: `curl <endpoint>`

当`pre-check`失败时，可以在日志中找到`HTTP-01`挑战的端点。
如果没有出现在日志中，您可以通过`kubectl`命令检查挑战 URL。

`<endpoint>` 是用于从证书`Issuer`测试 HTTP-01 的 URL。
以 Let's Encrypt 为例，URL 的格式为`<domain>/.well-known/acme-challenge/<hash>`

## 负载均衡器 HTTP 端点

如果您正在使用负载均衡器(在托管的 Kubernetes 服务之外)，您应该能够将负载均衡器协议配置为 HTTP, HTTPS, TCP, UDP。
一些负载均衡器现在通过 Let's Encrypt 提供免费 TLS 证书。

当您的负载均衡器使用 HTTP 协议时，它可以拦截挑战 URL，用它们的哈希替换响应的验证哈希。

在这种情况下，cert-manager 将失败 `did not get expected response when querying endpoint, expected 'xxxx' but got: yyyy (truncated)`.

由于多种原因，可以抛出这种错误。
这个案例显示了一个格式正确的响应，但不是预期的响应。
解决方案是使用 TCP 协议配置负载均衡器，这样 HTTP 请求就不会被主机拦截。
