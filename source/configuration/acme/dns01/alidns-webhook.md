# alidns-webhook

[![alidns-webhook](https://github-readme-stats.vercel.app/api/pin/?username=pragkent&repo=alidns-webhook)](https://github.com/pragkent/alidns-webhook)

用于 alidns 的 ACME DNS webhook 提供商。

## 安装证书管理器

请在此查找文件: https://cert-manager.io/docs/installation/kubernetes/

## 安装 webhook (cert-manager(v0.11) 及以上)

1.  安装 alidns-webhook

    ```bash
    # Install alidns-webhook to cert-manager namespace.
    kubectl apply -f https://raw.githubusercontent.com/pragkent/alidns-webhook/master/deploy/bundle.yaml
    ```

2.  创建包含 alidns 凭证的秘密

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: alidns-secret
      namespace: cert-manager
    data:
      access-key: YOUR_ACCESS_KEY
      secret-key: YOUR_SECRET_KEY
    ```

3.  发行人例子

    ```yaml
    apiVersion: cert-manager.io/v1alpha2
    kind: ClusterIssuer
    metadata:
      name: letsencrypt-staging
    spec:
      acme:
        # Change to your letsencrypt email
        email: certmaster@example.com
        server: https://acme-staging-v02.api.letsencrypt.org/directory
        privateKeySecretRef:
          name: letsencrypt-staging-account-key
        solvers:
          - dns01:
              webhook:
              groupName: acme.yourcompany.com
              solverName: alidns
              config:
                region: ""
                accessKeySecretRef:
                  name: alidns-secret
                  key: access-key
                secretKeySecretRef:
                  name: alidns-secret
                  key: secret-key
    ```

4.  颁发证书

    ```yaml
    apiVersion: cert-manager.io/v1alpha2
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
        name: letsencrypt-staging
        kind: ClusterIssuer
    ```

## 安装 webhook (cert-manager(v0.11) 之前的)

1.  安装 alidns-webhook

    ```bash
    # Install alidns-webhook to cert-manager namespace.
    kubectl apply -f https://raw.githubusercontent.com/pragkent/alidns-webhook/master/deploy/legacy.yaml
    ```

2.  创建包含 alidns 凭证的秘密

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: alidns-secret
      namespace: cert-manager
    data:
      access-key: YOUR_ACCESS_KEY
      secret-key: YOUR_SECRET_KEY
    ```

3.  发行人例子

    ```yaml
    apiVersion: certmanager.k8s.io/v1alpha1
    kind: ClusterIssuer
    metadata:
      name: letsencrypt-staging
    spec:
      acme:
      email: certmaster@example.com
      server: https://acme-staging-v02.api.letsencrypt.org/directory
      privateKeySecretRef:
      name: letsencrypt-staging-account-key
      solvers:
        - dns01:
            webhook:
                groupName: acme.yourcompany.com
                solverName: alidns
                config:
                    region: ""
                    accessKeySecretRef:
                    name: alidns-secret
                    key: access-key
                    secretKeySecretRef:
                    name: alidns-secret
                    key: secret-key
    ```

4.  颁发证书

    ```yaml
    apiVersion: certmanager.k8s.io/v1alpha1
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
        name: letsencrypt-staging
        kind: ClusterIssuer
    ```
