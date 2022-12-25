# 阿里云 DNS ACME webhook

[![cert-manager-alidns-webhook](https://github-readme-stats.vercel.app/api/pin/?username=DEVmachine-fr&repo=cert-manager-alidns-webhook)](https://github.com/DEVmachine-fr/cert-manager-alidns-webhook)

[Alibaba Cloud DNS ACME webhook](https://github.com/DEVmachine-fr/cert-manager-alidns-webhook)

该项目基于最初在<https://github.com/go-acme/lego>中提交的代码

这是一个网络钩子实现的 Cert-Manager 使用阿里巴巴云 DNS(又名 AliDNS)。
有关 webhook 的更多细节，请参阅 cert-manager 的文档: <https://cert-manager.io/docs/concepts/webhook/>

## 安装

```bash
helm repo add cert-manager-alidns-webhook https://devmachine-fr.github.io/cert-manager-alidns-webhook
helm repo update
helm install cert-manager-alidns-webhook/alidns-webhook
```

创建持有阿里巴巴凭证的秘密:

```bash
kubectl create secret generic alidns-secrets --from-literal="access-token=yourtoken" --from-literal="secret-key=yoursecretkey"
```

## 创建颁发者

要使用的求解器的名称是`alidns-solver`。
您可以通过如下方式创建发行人:

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt
  namespace: default
spec:
  acme:
    email: contact@example.com
    privateKeySecretRef:
      name: letsencrypt
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    solvers:
      - dns01:
          webhook:
            config:
              accessTokenSecretRef:
                key: access-token
                name: alidns-secrets
              regionId: cn-beijing
              secretKeySecretRef:
                key: secret-key
                name: alidns-secrets
            groupName: example.com
            solverName: alidns-solver
        selector:
          dnsNames:
            - example.com
            - "*.example.com"
```

或者您可以创建一个`ClusterIssuer`，如下所示:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    email: contact@example.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt
    solvers:
      - dns01:
          webhook:
            config:
              accessTokenSecretRef:
                key: access-token
                name: alidns-secrets
              regionId: cn-beijing
              secretKeySecretRef:
                key: secret-key
                name: alidns-secrets
            groupName: example.com
            solverName: alidns-solver
```

有关更多信息，请参阅证书管理器文档: <https://cert-manager.io/docs/configuration/acme/dns01/>

## 创建认证

然后创建将使用该颁发者的证书: <https://cert-manager.io/docs/usage/certificate/>

使用 Issuer 创建认证，如下所示:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-tls
spec:
  secretName: example-com-tls
  commonName: example.com
  dnsNames:
    - example.com
    - "*.example.com"
  issuerRef:
    name: letsencrypt
    kind: Issuer
```

或使用 ClusterIssuer 创建认证，如下所示:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-tls
spec:
  secretName: example-com-tls
  commonName: example.com
  dnsNames:
    - example.com
    - "*.example.com"
  issuerRef:
    name: letsencrypt
    kind: ClusterIssuer
```
