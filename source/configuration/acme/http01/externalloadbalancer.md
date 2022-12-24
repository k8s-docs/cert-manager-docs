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

1. Check that the endpoint of the challenge is accessible to the public : `curl <endpoint>`
2. Check that the challenge endpoint is NOT accessible from inside behind the Load Balancer: use SSH to open a session on a node places behind the LB; then launch the same command than before : `curl <endpoint>`

The `HTTP-01` challenge's endpoint can be found in the logs when the `pre-check` fails. If it does not appear in the logs, you can check the challenge URL by `kubectl`command.

`<endpoint>` is the URL used to test the HTTP-01 from the certificate `Issuer`. For Let's Encrypt for example, the URL is formed like `<domain>/.well-known/acme-challenge/<hash>`

## 负载均衡器 HTTP 端点

If you are using a Load Balancer (outside a managed Kubernetes service), you should be able to configure the Load Balancer protocol as HTTP, HTTPS, TCP, UDP. Several Load Balancer now offer free TLS certificates with Let's Encrypt.

When using HTTP(s) protocols for your Load Balancer, it can intercept the challenge URL to replace the response's verification hash with their hash.

In this case, cert-manager will fail `did not get expected response when querying endpoint, expected 'xxxx' but got: yyyy (truncated)`.

This kind of error can be thrown for multiple reasons. This case shows a correctly formatted response, but not the expected one. The solution is to configure the Load Balancer with TCP protocol so that the HTTP request will not be intercepted by the host.
