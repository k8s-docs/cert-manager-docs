---
title: "ACMEDNS"
linkTitle: "ACMEDNS"
weight: 30
type: "docs"
---

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: example-issuer
spec:
  acme:
    ...
    solvers:
    - dns01:
        acmedns:
          host: https://acme.example.com
          accountSecretRef:
            name: acme-dns
            key: acmedns.json
```

一般来说, 客户 ACMEDNS 上的用户代表进行登记，并告知他们，他们必须创建 CNAME 条目.
这是不可能的证书管理器，它是一个非交互系统.
登记必须预先进行，并且所得的凭证 JSON 上传到群集作为`Secret`.
在这个例子中，我们使用`curl`和直接的 API 端点.
有关设置和配置 ACMEDNS 信息可在[ACMEDNS 项目页](https://github.com/joohoi/acme-dns)上.

1. 首先，注册与 ACMEDNS 服务器，在这个例子中，有一个在`auth.example.com`运行

`curl -X POST http://auth.example.com/register` 将返回证书的 JSON 您的注册:

```json
{
  "username": "eabcdb41-d89f-4580-826f-3e62e9755ef2",
  "password": "pbAXVjlIOE01xbut7YnAbkhMQIkcwoHO0ek2j4Q0",
  "fulldomain": "d420c923-bbd7-4056-ab64-c3ca54c9b3cf.auth.example.com",
  "subdomain": "d420c923-bbd7-4056-ab64-c3ca54c9b3cf",
  "allowfrom": []
}
```

It is strongly recommended to restrict the update endpoint to the IP range of your pods.
This is done at registration time as follows:

`curl -X POST http://auth.example.com/register -H "Content-Type: application/json" --data '{"allowfrom": ["10.244.0.0/16"]}'`

Make sure to update the `allowfrom` field to match your cluster configuration. The JSON will now look like

```json
{
  "username": "eabcdb41-d89f-4580-826f-3e62e9755ef2",
  "password": "pbAXVjlIOE01xbut7YnAbkhMQIkcwoHO0ek2j4Q0",
  "fulldomain": "d420c923-bbd7-4056-ab64-c3ca54c9b3cf.auth.example.com",
  "subdomain": "d420c923-bbd7-4056-ab64-c3ca54c9b3cf",
  "allowfrom": ["10.244.0.0/16"]
}
```

2. Save this JSON to a file with the key as your domain. You can specify
   multiple domains with the same credentials if you like. In our example, the
   returned credentials can be used to verify ownership of `example.com` and and
   `example.org`.

```json
{
  "example.com": {
    "username": "eabcdb41-d89f-4580-826f-3e62e9755ef2",
    "password": "pbAXVjlIOE01xbut7YnAbkhMQIkcwoHO0ek2j4Q0",
    "fulldomain": "d420c923-bbd7-4056-ab64-c3ca54c9b3cf.auth.example.com",
    "subdomain": "d420c923-bbd7-4056-ab64-c3ca54c9b3cf",
    "allowfrom": ["10.244.0.0/16"]
  },
  "example.org": {
    "username": "eabcdb41-d89f-4580-826f-3e62e9755ef2",
    "password": "pbAXVjlIOE01xbut7YnAbkhMQIkcwoHO0ek2j4Q0",
    "fulldomain": "d420c923-bbd7-4056-ab64-c3ca54c9b3cf.auth.example.com",
    "subdomain": "d420c923-bbd7-4056-ab64-c3ca54c9b3cf",
    "allowfrom": ["10.244.0.0/16"]
  }
}
```

3. Next, update your primary DNS server with the CNAME record that will tell the
   verifier how to locate the challenge TXT record. This is obtained from the
   `fulldomain` field in the registration:

```
_acme-challenge.example.com CNAME d420c923-bbd7-4056-ab64-c3ca54c9b3cf.auth.example.com
_acme-challenge.example.org CNAME d420c923-bbd7-4056-ab64-c3ca54c9b3cf.auth.example.com
```

> Note: that the "name" of the record always has the `_acme-challenge`
> subdomain, and the "value" of the record matches exactly the `fulldomain`
> field from registration.

At verification time, the domain name `d420c923-bbd7-4056-ab64-c3ca54c9b3cf.auth.example.com` will be a TXT
record that is set to your validation token. When the verifier queries `_acme-challenge.example.com`, it will
be directed to the correct location by this CNAME record. This proves that you control `example.com`

4. Create a secret from the credentials JSON that was saved in step 2, this secret is referenced
   in the `accountSecretRef` field of your DNS01 issuer settings.
   When creating an `Issuer` both this `Issuer` and `Secret` must be in the same namespace.
   However for a `ClusterIssuer` (which does not have a namespace) the `Secret` must be placed in
   the same namespace as where the cert-manager pod is running in (in the default setup `cert-manager`).

```bash
$ kubectl create secret generic acme-dns --from-file acmedns.json
```
