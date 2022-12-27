# DNS01

## 配置 DNS01 挑战提供程序

本页包含有关`Issuer`源的 DNS01 挑战解决程序配置上可用的不同选项的详细信息。

有关配置 ACME`Issuers` 及其 API 格式的更多信息，请阅读[ACME 发行者](../README.md)文档。

DNS01 提供程序配置必须在`Issuer`源上指定，类似于设置文档中的示例。

您可以在[Let's Encrypt 挑战类型页面](https://letsencrypt.org/docs/challenge-types/#dns-01-challenge)上阅读有关 DNS01 挑战类型的工作原理.

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: example-issuer
spec:
  acme:
    email: user@example.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: example-issuer-account-key
    solvers:
      - dns01:
          cloudDNS:
            project: my-project
            serviceAccountSecretRef:
              name: prod-clouddns-svc-acct-secret
              key: service-account.json
```

每个发行者可以指定多个不同的 DNS01 质询提供者，也可以在一个`Issuer`上拥有同一个 DNS 提供者的多个实例(例如，可以设置两个 CloudDNS 帐户，每个帐户都有自己的名称)。

有关在单个`Issuer`上使用多个求解器类型的更多信息，请阅读多求解器类型一节。

## 设置 DNS01 自检的名称服务器

在尝试 DNS01 挑战之前，cert-manager 将检查是否存在正确的 DNS 记录。
默认情况下，cert-manager 将使用从`/etc/resolv.conf`中获得的递归名称服务器来查询权威名称服务器，然后它将直接查询以验证 DNS 记录的存在。

如果这不是所希望的(例如使用多个权威名称服务器或分割地平线的 DNS)， cert-manager 控制器会暴露两个标志，允许您更改此行为:

`--dns01-recursive-nameservers` cert-manager 应该查询的递归名称服务器的主机和端口的逗号分隔字符串。

`--dns01-recursive-nameservers-only` 强制 cert-manager 仅使用递归名称服务器进行验证。
启用此选项可能会导致 DNS01 自我检查花费更长的时间，因为递归名称服务器执行缓存。

!!! Example usage:

    ```bash
    --dns01-recursive-nameservers-only --dns01-recursive-nameservers=8.8.8.8:53,1.1.1.1:53
    ```

如果你正在使用`cert-manager` helm chart，你可以通过`.Values.extraArgs`或在 helm 安装/升级时使用`--set`命令设置递归名称服务器:

```bash
--set 'extraArgs={--dns01-recursive-nameservers-only,--dns01-recursive-nameservers=8.8.8.8:53\,1.1.1.1:53}'
```

## DNS01 的委托域

缺省情况下，cert-manager 不会跟踪指向子域的 CNAME 记录。

如果不希望授予 cert-manager 对根 DNS 区域的访问权限，那么可以将`_acme-challenge.example.com`子域委托给其他一些特权较低的域(`less-privileged.example.org`)。
这可以通过以下方式实现。
比如说，一个有两个区域:

- `example.com`
- `less-privileged.example.org`

1. 创建一条 CNAME 记录，指向这个特权较低的域:

```
_acme-challenge.example.com	IN	CNAME	_acme-challenge.less-privileged.example.org.
```

2. 授予证书管理器更新特权较低的`less-privileged.example.org`区域的权限

3. 为更新这个特权较低的区域提供配置/凭据，并在相关的`dns01`求解器中添加一个额外的字段。
   注意，`selector`字段仍然适用于原来的`example.com`，而`less-privileged.example.org`则提供了凭据。

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  ...
spec:
  acme:
    ...
    solvers:
    - selector:
        dnsZones:
        - 'example.com'
      dns01:
        # Valid values are None and Follow
        cnameStrategy: Follow
        route53:
          region: eu-central-1
          accessKeyID: <Access ID for less-privileged.example.org here>
          hostedZoneID: <Zone ID for less-privileged.example.org here>
          secretAccessKeySecretRef:
            ...
```

如果您有许多(子)域需要单独的证书，则可以共享一个别名较低特权的域。
要实现它，应该为每个(子)域创建一个 CNAME 记录，如下所示:

```txt
_acme-challenge.example.com	    IN	CNAME	_acme-challenge.less-privileged.example.org.
_acme-challenge.www.example.com	IN	CNAME	_acme-challenge.less-privileged.example.org.
_acme-challenge.foo.example.com	IN	CNAME	_acme-challenge.less-privileged.example.org.
_acme-challenge.bar.example.com	IN	CNAME	_acme-challenge.less-privileged.example.org.
```

有了这个配置，cert-manager 将递归地跟踪 CNAME 记录，以确定在 DNS01 挑战期间要更新哪个 DNS 区域。

## 支持的 DNS01 提供商

ACME`Issuer`支持许多不同的 DNS 提供程序。
下面是可用的供应商列表，它们的`.yaml`配置，以及关于它们使用的其他 Kubernetes 和供应商特定注意事项。

- [ACMEDNS](./acme-dns.md)
- [Akamai](./akamai.md)
- [AzureDNS](./azuredns.md)
- [CloudFlare](./cloudflare.md)
- [Google](./google.md)
- [Route53](./route53.md)
- [DigitalOcean](./digitalocean.md)
- [RFC2136](./rfc2136.md)

## Webhook

cert-manager 还支持使用外部 webhook 的树外 DNS 提供商。
链接到这些受支持的提供商及其文档如下:

- [`AliDNS-Webhook`](https://github.com/pragkent/alidns-webhook)
- [`cert-manager-alidns-webhook`](https://github.com/DEVmachine-fr/cert-manager-alidns-webhook)
- [`cert-manager-webhook-civo`](https://github.com/okteto/cert-manager-webhook-civo)
- [`cert-manager-webhook-dnspod`](https://github.com/qqshfox/cert-manager-webhook-dnspod)
- [`cert-manager-webhook-dnsimple`](https://github.com/neoskop/cert-manager-webhook-dnsimple)
- [`cert-manager-webhook-gandi`](https://github.com/bwolf/cert-manager-webhook-gandi)
- [`cert-manager-webhook-infomaniak`](https://github.com/Infomaniak/cert-manager-webhook-infomaniak)
- [`cert-manager-webhook-inwx`](https://gitlab.com/smueller18/cert-manager-webhook-inwx)
- [`cert-manager-webhook-linode`](https://github.com/slicen/cert-manager-webhook-linode)
- [`cert-manager-webhook-oci`](https://gitlab.com/dn13/cert-manager-webhook-oci) (Oracle Cloud Infrastructure)
- [`cert-manager-webhook-scaleway`](https://github.com/scaleway/cert-manager-webhook-scaleway)
- [`cert-manager-webhook-selectel`](https://github.com/selectel/cert-manager-webhook-selectel)
- [`cert-manager-webhook-softlayer`](https://github.com/cgroschupp/cert-manager-webhook-softlayer)
- [`cert-manager-webhook-ibmcis`](https://github.com/jb-dk/cert-manager-webhook-ibmcis)
- [`cert-manager-webhook-loopia`](https://github.com/Identitry/cert-manager-webhook-loopia)
- [`cert-manager-webhook-arvan`](https://github.com/kiandigital/cert-manager-webhook-arvan)
- [`bizflycloud-certmanager-dns-webhook`](https://github.com/bizflycloud/bizflycloud-certmanager-dns-webhook)
- [`cert-manager-webhook-hetzner`](https://github.com/vadimkim/cert-manager-webhook-hetzner)
- [`cert-manager-webhook-yandex-cloud`](https://github.com/malinink/cert-manager-webhook-yandex-cloud)
- [`cert-manager-webhook-netcup`](https://github.com/aellwein/cert-manager-webhook-netcup)

你可以在[这里](./webhook.md)找到更多关于如何配置 webhook 提供者的信息。

要创建一个新的不受支持的 DNS 提供程序，请参考开发文档[此处](../../../contributing/dns-providers.md).
