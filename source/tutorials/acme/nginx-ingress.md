---
title: 护卫 NGINX-ingress
description: "cert-manager tutorials: 使用ingress-nginx解决ACME HTTP-01挑战"
---

# 护卫 NGINX-ingress

本教程将详细介绍如何使用 NGINX 安装和保护集群的入口。

## 步骤 1 - 安装 Helm

> _如果你安装了 helm，跳过这一节。_

安装`cert-manager` 最简单的方法是使用[`Helm`](https://helm.sh)，这是 Kubernetes 源的模板和部署工具。

首先，确保 Helm 客户端按照[Helm 安装说明](https://helm.sh/docs/intro/install/)进行安装.

!!! example "on MacOS"

    ```bash
    brew install kubernetes-helm
    ```

## 步骤 2 - 部署 NGINX 入口控制器

一个[`kubernetes入口控制器`](https://kubernetes.io/docs/concepts/services-networking/ingress)被设计为 HTTP 和 HTTPS 流量的接入点，以访问集群中运行的软件。
`ingress-nginx-controller` 通过提供由云提供商的负载均衡器支持的 HTTP 代理服务来实现这一点。

你可以从[关于`ingress-nginx`的文档](https://kubernetes.github.io/ingress-nginx/)中获得更多关于`ingress-nginx`及其工作原理的详细信息。

为 ingress-nginx 添加最新的 helm 存储库

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```

用最新的图表更新 helm 库:

```bash
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "stable" chart repository
...Successfully got an update from the "ingress-nginx" chart repository
...Successfully got an update from the "coreos" chart repository
Update Complete. ⎈ Happy Helming!⎈
```

使用`helm`安装 NGINX Ingress 控制器:

```bash
$ helm install quickstart ingress-nginx/ingress-nginx

NAME: quickstart
... lots of output ...
```

云提供商提供并链接一个公共 IP 地址可能需要一到两分钟。
当它完成后，您可以使用 `kubectl` 命令看到外部 IP 地址:

```bash
$ kubectl get svc
NAME                                            TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
kubernetes                                      ClusterIP      10.0.0.1       <none>        443/TCP                      13m
quickstart-ingress-nginx-controller             LoadBalancer   10.0.114.241   <pending>     80:31635/TCP,443:30062/TCP   8m16s
quickstart-ingress-nginx-controller-admission   ClusterIP      10.0.188.24    <none>        443/TCP                      8m16s
```

该命令显示集群中的所有服务(在`default`名称空间中)，以及它们拥有的任何外部 IP 地址。
当您第一次创建控制器时，您的云提供商还没有通过`LoadBalancer`分配 IP 地址。
在此之前，服务的外部 IP 地址将被列为`<pending>`。

您的云提供商可以选择在创建入口控制器之前保留一个 IP 地址，并使用该 IP 地址，而不是从池中分配 IP 地址。
请阅读您的云提供商的文档，了解如何安排这一点。

## 步骤 3 - 分配 DNS 名称

分配给入口控制器的外部 IP 是所有传入流量都应该路由到的 IP。
要启用此功能，请将其添加到您控制的 DNS 区域，例如`www.example.com`。

这个快速入门假设您知道如何将 DNS 条目分配给 IP 地址，并且会这样做。

## 步骤 4 - 部署示例服务

您的服务可以有自己的图表，也可以直接使用清单部署它。
本快速入门使用清单创建和公开示例服务。
示例服务使用[`kuard`](https://github.com/kubernetes-up-and-running/kuard)，一个演示应用程序。

快速启动示例为示例使用了三个清单。
前两个是一个示例部署和一个关联的服务:

```yaml title="./example/deployment.yaml"
--8<-- "deployment.yaml"
```

```yaml title="./example/service.yaml"
--8<-- "service.yaml"
```

您可以在本地创建、下载和引用这些文件，也可以从本文档的 GitHub 源存储库引用它们。
要直接从 GitHub 安装教程文件中的示例服务，请执行以下操作:

```bash
kubectl apply -f https://raw.githubusercontent.com/cert-manager/website/master/content/docs/tutorials/acme/example/deployment.yaml
# expected output: deployment.extensions "kuard" created

kubectl apply -f https://raw.githubusercontent.com/cert-manager/website/master/content/docs/tutorials/acme/example/service.yaml
# expected output: service "kuard" created
```

Kubernetes 使用[Ingress 源](https://kubernetes.io/docs/concepts/services-networking/ingress/)在集群外部公开这个示例服务。
您需要下载并修改示例清单，以反映您拥有或控制的域，以完成此示例。

你可以从一个示例入口开始:

```yaml title="./example/ingress.yaml"
--8<-- "ingress.yaml"
```

您可以从 GitHub 下载清单示例，编辑它，并使用下面的命令将清单提交到 Kubernetes。
在编辑器中编辑文件，保存后:

```bash
kubectl create --edit -f https://raw.githubusercontent.com/cert-manager/website/master/content/docs/tutorials/acme/example/ingress.yaml
# expected output: ingress.extensions "kuard" created
```

!!! note

    我们上面展示的ingress示例中有一个`host`定义。
    当请求的主机名与入口中的定义匹配时，`ingress-nginx-controller`将路由流量。
    您可以在规则中部署一个没有“主机”定义的入口，但该模式不适用于TLS证书，因为TLS证书需要完全限定的域名。

一旦它被部署，你可以使用命令`kubectl get ingress`来查看入口的状态:

```text
NAME      HOSTS     ADDRESS   PORTS     AGE
kuard     *                   80, 443   17s
```

完全创建入口可能需要几分钟时间，具体时间取决于您的服务提供商。
当它被创建并链接到位时，入口也会显示一个地址:

```text
NAME      HOSTS     ADDRESS         PORTS     AGE
kuard     *         203.0.113.2   80        9m
```

!!! note

    ingress上的IP地址可能与`ingress-nginx-controller`拥有的IP地址不匹配。
    这很好，这是托管Kubernetes集群的服务提供者的一个怪癖/实现细节。
    由于我们使用`ingress-nginx-controller`而不是任何云提供商特定的入口后端，
    使用为`quickstart-ingress-nginx-controller ` `LoadBalancer`资源定义和分配的IP地址作为您服务的主要接入点。

确保您可以通过上面添加的域名访问该服务，例如 `http://www.example.com`。
最简单的方法是打开浏览器，输入您在 DNS 中设置的名称，我们刚刚为其添加了入口。

你也可以使用像 `curl` 这样的命令行工具来检查入口。

```bash
$ curl -kivL -H 'Host: www.example.com' 'http://203.0.113.2'
```

这个 curl 命令的选项将在任何重定向之后提供详细输出，在输出中显示 TLS 报头，并且不会在不安全的证书上出现错误。
使用`ingress-nginx-controller`，该服务将使用 TLS 证书，但它将使用`ingress-nginx-controller`提供的自签名证书作为默认值。
浏览器将显示一个警告，这是一个无效的证书。这是预期的，也是正常的，因为我们还没有使用证书管理器为我们的站点获取完全受信任的证书。

!!! Warning

    确保你的入口是可用的，并在互联网上正确响应是至关重要的。
    这个快速入门示例使用Let’s Encrypt来提供证书，它期望并验证服务可用，
    并且在颁发证书的过程中使用该验证作为对域的请求属于对域有足够控制的人的证明。

## 步骤 5 - 部署 cert-manager

我们需要安装 cert-manager 来完成 Kubernetes 的工作，以请求证书并响应验证它的挑战。
我们可以使用 Helm 或普通的 Kubernetes 清单来安装 cert-manager。

因为我们之前安装了 Helm，我们会假设你想要使用 Helm;按照 [Helm guide](../../installation/helm.md)。
其他方法请参考 cert-manager 的[安装文档](../../installation/README.md)。

cert-manager 主要使用两种不同的自定义 Kubernetes 资源(称为[`CRDs`](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/))来配置和控制它的操作方式，以及存储状态。
这些资源是颁发者和证书。

### Issuers(颁发者)

颁发者定义了证书管理器 _如何_ 请求 TLS 证书。
颁发者在 Kubernetes 中是特定于单个名称空间的，但也有一个`ClusterIssuer` ，它意味着是一个集群范围的版本。

注意确保您的颁发者与要创建的证书在相同的名称空间中创建。
您可能需要在`kubectl create`命令中添加`-n my-namespace` 。

你的另一个选择是用`ClusterIssuers`替换你的`issuer`;`ClusterIssuers`源适用于集群中的所有 Ingress 资源。
如果使用`ClusterIssuers`，请记住将 Ingress 注释`cert-manager.io/issuer` 更新为 `cert-manager.io/cluster-issuer`。

如果您发现颁发者有问题，请遵循[故障排除颁发 ACME 证书](../../troubleshooting/acme.md)指南。

关于`Issuers` 和 `ClusterIssuers`之间的区别的更多信息 - 包括您可以在[颁发者概念](../../concepts/issuer.md#namespaces)上找到它们。

### Certificate(证书)

证书资源允许您指定要请求的证书的详细信息。
它们引用颁发者来定义它们将*如何*发行。

有关更多信息，请参见[证书概念](../../concepts/certificate.md).

## 步骤 6 - 配置一个 Let's Encrypt Issuer

在本例中，我们将为 Let’s Encrypt 设置两个颁发者:staging 和 production。

Let's Encrypt 产品发行方有[非常严格的速率限制](https://letsencrypt.org/docs/rate-limits/)。
当你在尝试和学习时，很容易就会触及这些极限。
由于存在这种风险，我们将从 Let's Encrypt staging 发行方开始，一旦我们对它的工作感到满意，我们将切换到 production 发行方。

注意，您将看到来自预演颁发者的关于不受信任证书的警告，但这是完全预期的。

在本地创建这个定义，并将电子邮件地址更新为您自己的。
此电子邮件是 Let's Encrypt 要求的，用于通知您证书过期和更新。

```yaml title="./example/staging-issuer.yaml"
--8<-- "staging-issuer.yaml"
```

编辑完成后，应用自定义资源:

```bash
kubectl create --edit -f https://raw.githubusercontent.com/cert-manager/website/master/content/docs/tutorials/acme/example/staging-issuer.yaml
# expected output: issuer.cert-manager.io "letsencrypt-staging" created
```

还要创建一个生产颁发者并部署它。
与预演发布者一样，您需要更新此示例并添加您自己的电子邮件地址。

```yaml title="./example/production-issuer.yaml"
--8<-- "production-issuer.yaml"
```

```bash
kubectl create --edit -f https://raw.githubusercontent.com/cert-manager/website/master/content/docs/tutorials/acme/example/production-issuer.yaml
# expected output: issuer.cert-manager.io "letsencrypt-prod" created
```

这两个颁发者都被配置为使用[`HTTP01`](../../configuration/acme/http01/README.md)挑战提供程序。

创建颁发者后检查它的状态:

```bash
$ kubectl describe issuer letsencrypt-staging
Name:         letsencrypt-staging
Namespace:    default
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"cert-manager.io/v1","kind":"Issuer","metadata":{"annotations":{},"name":"letsencrypt-staging","namespace":"default"},(...)}
API Version:  cert-manager.io/v1
Kind:         Issuer
Metadata:
  Cluster Name:
  Creation Timestamp:  2018-11-17T18:03:54Z
  Generation:          0
  Resource Version:    9092
  Self Link:           /apis/cert-manager.io/v1/namespaces/default/issuers/letsencrypt-staging
  UID:                 25b7ae77-ea93-11e8-82f8-42010a8a00b5
Spec:
  Acme:
    Email:  email@example.com
    Private Key Secret Ref:
      Key:
      Name:  letsencrypt-staging
    Server:  https://acme-staging-v02.api.letsencrypt.org/directory
    Solvers:
      Http 01:
        Ingress:
          Class:  nginx
Status:
  Acme:
    Uri:  https://acme-staging-v02.api.letsencrypt.org/acme/acct/7374163
  Conditions:
    Last Transition Time:  2018-11-17T18:04:00Z
    Message:               The ACME account was registered with the ACME server
    Reason:                ACMEAccountRegistered
    Status:                True
    Type:                  Ready
Events:                    <none>
```

您应该看到颁发者列出了一个注册帐户。

## 步骤 7 - 部署 TLS 入口资源

所有必要的配置就绪之后，我们现在就可以开始请求 TLS 证书了。
主要有两种方法:在入口上使用[`ingress-shim`](../../usage/ingress.md)注释或直接创建证书资源。

在本例中，我们将向入口添加注释，并利用 ingress-shim 让它代表我们创建证书资源。
创建证书后，证书管理器将更新或创建入口资源，并使用该资源验证域。
验证和颁发后，证书管理器将创建或更新证书中定义的秘密。

!!! Note

    在入口中使用的秘密应该与证书中定义的秘密相匹配。
    没有任何显式的检查，所以输入错误将导致`ingress-nginx-controller`回落到它的自签名证书。
    在我们的示例中，我们在ingress(和ingress-shim)上使用注释，它们将为您创建正确的秘密。

编辑入口，添加在前面例子中被注释掉的注释:

```yaml title="./example/ingress-tls.yaml"
--8<-- "ingress-tls.yaml"
```

应用它:

```bash
kubectl create --edit -f https://raw.githubusercontent.com/cert-manager/website/master/content/docs/tutorials/acme/example/ingress-tls.yaml
# expected output: ingress.extensions "kuard" configured
```

证书管理器将读取这些注释并使用它们创建一个证书，您可以请求并查看:

```bash
$ kubectl get certificate
NAME                     READY   SECRET                   AGE
quickstart-example-tls   True    quickstart-example-tls   16m
```

证书管理器反映证书对象中每个请求的进程状态。
您可以使用`kubectl describe`命令查看该信息:

```bash
$ kubectl describe certificate quickstart-example-tls
Name:         quickstart-example-tls
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         Certificate
Metadata:
  Cluster Name:
  Creation Timestamp:  2018-11-17T17:58:37Z
  Generation:          0
  Owner References:
    API Version:           networking.k8s.io/v1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  Ingress
    Name:                  kuard
    UID:                   a3e9f935-ea87-11e8-82f8-42010a8a00b5
  Resource Version:        9295
  Self Link:               /apis/cert-manager.io/v1/namespaces/default/certificates/quickstart-example-tls
  UID:                     68d43400-ea92-11e8-82f8-42010a8a00b5
Spec:
  Dns Names:
    www.example.com
  Issuer Ref:
    Kind:       Issuer
    Name:       letsencrypt-staging
  Secret Name:  quickstart-example-tls
Status:
  Acme:
    Order:
      URL:  https://acme-staging-v02.api.letsencrypt.org/acme/order/7374163/13665676
  Conditions:
    Last Transition Time:  2018-11-17T18:05:57Z
    Message:               Certificate issued successfully
    Reason:                CertIssued
    Status:                True
    Type:                  Ready
Events:
  Type     Reason          Age                From          Message
  ----     ------          ----               ----          -------
  Normal   CreateOrder     9m                 cert-manager  Created new ACME order, attempting validation...
  Normal   DomainVerified  8m                 cert-manager  Domain "www.example.com" verified with "http-01" validation
  Normal   IssueCert       8m                 cert-manager  Issuing certificate...
  Normal   CertObtained    7m                 cert-manager  Obtained certificate from ACME server
  Normal   CertIssued      7m                 cert-manager  Certificate issued Successfully
```

与此资源相关并列在`describe`结果底部的事件显示了请求的状态。
在上面的示例中，证书在几分钟内被验证并颁发。

完成后，证书管理器将根据入口资源中使用的秘密创建一个包含证书详细信息的秘密。
你也可以使用 describe 命令来查看一些细节:

```bash
$ kubectl describe secret quickstart-example-tls
Name:         quickstart-example-tls
Namespace:    default
Labels:       cert-manager.io/certificate-name=quickstart-example-tls
Annotations:  cert-manager.io/alt-names=www.example.com
              cert-manager.io/common-name=www.example.com
              cert-manager.io/issuer-kind=Issuer
              cert-manager.io/issuer-name=letsencrypt-staging

Type:  kubernetes.io/tls

Data
====
tls.crt:  3566 bytes
tls.key:  1675 bytes
```

现在我们有信心所有的配置都是正确的，你可以更新入口中的注释来指定生产颁发者:

```yaml title="./example/ingress-tls-final.yaml"
--8<-- "ingress-tls-final.yaml"
```

```bash
$ kubectl create --edit -f https://raw.githubusercontent.com/cert-manager/website/master/content/docs/tutorials/acme/example/ingress-tls-final.yaml
ingress.extensions "kuard" configured
```

您还需要删除现有的秘密，证书管理器正在监视该秘密，并将使它使用更新后的颁发者重新处理请求。

```bash
$ kubectl delete secret quickstart-example-tls
secret "quickstart-example-tls" deleted
```

这将启动获取新证书的过程，使用 describe 可以查看状态。
在更新了生产证书之后，您应该看到示例 KUARD 在您的域中运行，其中包含已签名的 TLS 证书。

```bash
$ kubectl describe certificate quickstart-example-tls
Name:         quickstart-example-tls
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         Certificate
Metadata:
  Cluster Name:
  Creation Timestamp:  2018-11-17T18:36:48Z
  Generation:          0
  Owner References:
    API Version:           networking.k8s.io/v1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  Ingress
    Name:                  kuard
    UID:                   a3e9f935-ea87-11e8-82f8-42010a8a00b5
  Resource Version:        283686
  Self Link:               /apis/cert-manager.io/v1/namespaces/default/certificates/quickstart-example-tls
  UID:                     bdd93b32-ea97-11e8-82f8-42010a8a00b5
Spec:
  Dns Names:
    www.example.com
  Issuer Ref:
    Kind:       Issuer
    Name:       letsencrypt-prod
  Secret Name:  quickstart-example-tls
Status:
  Conditions:
    Last Transition Time:  2019-01-09T13:52:05Z
    Message:               Certificate does not exist
    Reason:                NotFound
    Status:                False
    Type:                  Ready
Events:
  Type    Reason        Age   From          Message
  ----    ------        ----  ----          -------
  Normal  Generated     18s   cert-manager  Generated new private key
  Normal  OrderCreated  18s   cert-manager  Created Order resource "quickstart-example-tls-889745041"
```

您可以通过在证书管理器为您的证书创建的订单资源上运行`kubectl describe`来查看 ACME 订单的当前状态:

```bash
$ kubectl describe order quickstart-example-tls-889745041
...
Events:
  Type    Reason      Age   From          Message
  ----    ------      ----  ----          -------
  Normal  Created     90s   cert-manager  Created Challenge resource "quickstart-example-tls-889745041-0" for domain "www.example.com"
```

在这里，我们可以看到 cert-manager 创建了 1 个'Challenge'源来完成订单。
您可以通过在自动创建的挑战源上运行`kubectl describe` 来深入了解当前 ACME 挑战的状态:

```bash
$ kubectl describe challenge quickstart-example-tls-889745041-0
...
Status:
  Presented:   true
  Processing:  true
  Reason:      Waiting for http-01 challenge propagation
  State:       pending
Events:
  Type    Reason     Age   From          Message
  ----    ------     ----  ----          -------
  Normal  Started    15s   cert-manager  Challenge scheduled for processing
  Normal  Presented  14s   cert-manager  Presented challenge using http-01 challenge mechanism
```

从上面，我们可以看到挑战已经'presented'，cert-manager 正在等待挑战记录传播到入口控制器。
你应该留意挑战资源上的新事件，因为'success'事件应该在一分钟左右打印出来(取决于你的入口控制器更新规则的速度):

```bash
$ kubectl describe challenge quickstart-example-tls-889745041-0
...
Status:
  Presented:   false
  Processing:  false
  Reason:      Successfully authorized domain
  State:       valid
Events:
  Type    Reason          Age   From          Message
  ----    ------          ----  ----          -------
  Normal  Started         71s   cert-manager  Challenge scheduled for processing
  Normal  Presented       70s   cert-manager  Presented challenge using http-01 challenge mechanism
  Normal  DomainVerified  2s    cert-manager  Domain "www.example.com" verified with "http-01" validation
```

!!! Note

    如果您的挑战没有变得'valid'，并且仍然处于'pending'状态(或进入'failed' 状态)，则很可能存在某种配置错误。
    阅读[挑战源参考文档](../../reference/api-docs.md#acme.cert-manager.io/v1.Challenge)了解更多关于调试失败挑战的信息。

一旦挑战完成，相应的挑战源将被删除，'Order'将被更新以反映 Order 的新状态:

```bash
$ kubectl describe order quickstart-example-tls-889745041
...
Events:
  Type    Reason      Age   From          Message
  ----    ------      ----  ----          -------
  Normal  Created     90s   cert-manager  Created Challenge resource "quickstart-example-tls-889745041-0" for domain "www.example.com"
  Normal  OrderValid  16s   cert-manager  Order completed successfully
```

最后，`Certificate`源将被更新以反映颁发过程的状态。
如果一切正常，你应该能够`describe`证书，并看到如下内容:

```bash
$ kubectl describe certificate quickstart-example-tls
Status:
  Conditions:
    Last Transition Time:  2019-01-09T13:57:52Z
    Message:               Certificate is up to date and has not expired
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2019-04-09T12:57:50Z
Events:
  Type    Reason         Age                  From          Message
  ----    ------         ----                 ----          -------
  Normal  Generated      11m                  cert-manager  Generated new private key
  Normal  OrderCreated   11m                  cert-manager  Created Order resource "quickstart-example-tls-889745041"
  Normal  OrderComplete  10m                  cert-manager  Order "quickstart-example-tls-889745041" completed successfully
```
