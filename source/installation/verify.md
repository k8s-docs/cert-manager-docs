# 安装验证

## 检查 cert-manager API

首先，确保[cmctl 已安装](../reference/cmctl.md#installation).

cmctl 对 Kubernetes 集群执行一个干运行证书创建检查。
如果成功，系统提示`The cert-manager API is ready`。

```bash
$ cmctl check api
The cert-manager API is ready
```

该命令也可以用于等待检查成功。
下面是在安装 cert-manager 的同时运行该命令的输出示例:

```bash
$ cmctl check api --wait=2m
Not ready: the cert-manager CRDs are not yet installed on the Kubernetes API server
Not ready: the cert-manager CRDs are not yet installed on the Kubernetes API server
Not ready: the cert-manager webhook deployment is not ready yet
Not ready: the cert-manager webhook deployment is not ready yet
Not ready: the cert-manager webhook deployment is not ready yet
Not ready: the cert-manager webhook deployment is not ready yet
The cert-manager API is ready
```

## 手动验证

安装了 cert-manager 之后，可以通过以下命令验证它是否正确部署
检查`cert-manager`命名空间是否正在运行 pod:

```bash
$ kubectl get pods --namespace cert-manager

NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-5c6866597-zw7kh               1/1     Running   0          2m
cert-manager-cainjector-577f6d9fd7-tr77l   1/1     Running   0          2m
cert-manager-webhook-787858fcdb-nlzsq      1/1     Running   0          2m
```

你应该看到`cert-manager`, `cert-manager-cainjector`, 和 `cert-manager-webhook` Pod 处于`Running`状态。
webhook 可能比其他 webhook 需要更长的时间才能成功提供。

如果遇到问题，请先查看[FAQ](../faq/README.md).

创建一个`Issuer`来测试 webhook 的工作情况。

```bash
$ cat <<EOF > test-resources.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager-test
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: test-selfsigned
  namespace: cert-manager-test
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: selfsigned-cert
  namespace: cert-manager-test
spec:
  dnsNames:
    - example.com
  secretName: selfsigned-cert-tls
  issuerRef:
    name: test-selfsigned
EOF
```

创建测试源。

```bash
$ kubectl apply -f test-resources.yaml
```

检查新创建的证书的状态。
在证书管理器处理证书请求之前，您可能需要等待几秒钟。

```bash
$ kubectl describe certificate -n cert-manager-test

...
Spec:
  Common Name:  example.com
  Issuer Ref:
    Name:       test-selfsigned
  Secret Name:  selfsigned-cert-tls
Status:
  Conditions:
    Last Transition Time:  2019-01-29T17:34:30Z
    Message:               Certificate is up to date and has not expired
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2019-04-29T17:34:29Z
Events:
  Type    Reason      Age   From          Message
  ----    ------      ----  ----          -------
  Normal  CertIssued  4s    cert-manager  Certificate issued successfully
```

清理测试源。

```bash
$ kubectl delete -f test-resources.yaml
```

如果以上所有步骤都正确地完成了，那么就可以开始了!

## 社区维护工具

您也可以通过社区维护[cert-manager-verifier](https://github.com/alenkacz/cert-manager-verifier)工具自动检查 cert-manager 配置是否正确。
