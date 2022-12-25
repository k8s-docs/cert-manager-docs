# 阿里云 K8s 集群部署 cert-manager [:link:](https://lxh.io/post/2022/kubenetes-aliyun-https/)

## 基础环境

- Kubenetes 版本: `v1.22.3`
- cert-manager 版本: `v1.8.1`
- 注意事项: Kubenetes 版本必须大于等于 `v1.19.0`

## 安装 cert-manager

```sh
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.8.1/cert-manager.yaml
```

检查是否安装成功:

```sh
$ cmctl check api
The cert-manager API is ready
# 出现如上情况表示安装成功
```

## 安装 alidns-webhook

```sh
$ helm repo add cert-manager-alidns-webhook https://devmachine-fr.github.io/cert-manager-alidns-webhook
$ helm repo update
$ helm install cert-manager-alidns-webhook/alidns-webhook
```

### 配置阿里云访问令牌

!!! note "配置的 access-key 必须拥有修改域名 DNS 的权限"

```sh
$ cat > secret.yaml << EOF
apiVersion: v1
kind: Secret
metadata:
  name: alidns-secret
  namespace: saas # 命名空间
data:
  access-token: YOUR_ACCESS_KEY # base64
  secret-key: YOUR_SECRET_KEY # base64
type: Opaque
EOF
$ kubectl apply -f issuer.yaml
```

或者使用下面这个

```sh
kubectl create secret generic alidns-secrets --from-literal="access-token=YOUR_ACCESS_KEY" --from-literal="secret-key=YOUR_SECRET_KEY" --namespace=saas
```

## 配置 issuer

官方文档说还支持 ClusterIssuer，但是我没成功，所以用了 issuer 的方式。

创建 issuer

```sh
$ cat > issuer.yaml << EOF
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt
  namespace: saas # 作用的命名空间，默认为`default`，必须和上面的alidns-secrets在同一命名空间
spec:
  acme:
    email: lxh@cxh.cn # 你的邮箱
    privateKeySecretRef:
      name: letsencrypt
    server: https://acme-v02.api.letsencrypt.org/directory
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
          groupName: example.com # 默认安装的值就是这个，照着抄就完事儿
          solverName: alidns-solver
      selector:
        dnsNames:
        - lxh.io # 你的域名
        - '*.lxh.io'
EOF
$ kubectl apply -f issuer.yaml
```

## 配置 certificate

```sh
$ cat > certificate.yaml << EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: saas-tls
  namespace: saas # 指定证书生成到哪个工作空间，必须和issuer的命名空间一致
spec:
  secretName: saas-tls #生成后证书的配置文件名称
  commonName: saas.xinli000.com # 证书的域名
  dnsNames:
  - saas.xinli000.com
  - "*.saas.xinli000.com"
  issuerRef:
    name: letsencrypt
    kind: Issuer
EOF
$ kubectl apply -f certificate.yaml
```

## 检查证书签发是否成功

证书签发正常情况下会在五分钟之内签发完毕，可以使用如下命令查看是否成功

```sh
$ kubectl get certificate -A
NAMESPACE      NAME                                    READY   SECRET                                  AGE
cert-manager   alidns-webhook-1654650790-ca            True    alidns-webhook-1654650790-ca            7d23h
cert-manager   alidns-webhook-1654650790-webhook-tls   True    alidns-webhook-1654650790-webhook-tls   7d23h
saas           saas-tls                                True    saas-tls                                7d22h
# 出现如上内容即为签发成功
```
