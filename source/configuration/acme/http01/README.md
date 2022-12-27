# HTTP01

!!! tip "📌 本页重点介绍如何解决 ACME HTTP-01 挑战。"

    如果您正在寻找如何通过注释入口源或网关源来自动创建证书源，请参见[保护入口源](../../../usage/ingress.md)和[保护网关源](../../../usage/gateway.md)。

cert-manager 使用您现有的入口或网关配置来解决 HTTP01 的挑战。

## 配置 HTTP01 入口求解器

本页包含有关`Issuer`源的 HTTP01 挑战解决程序配置上可用的不同选项的详细信息。
有关配置 ACME 发行者及其 API 格式的更多信息，请阅读[ACME 发行者](../README.md)文档。

您可以在[Let's Encrypt challenge 类型页面](https://letsencrypt.org/docs/challenge-types/#http-01-challenge)上阅读有关 HTTP01 挑战类型如何工作的内容.

下面是一个简单的`HTTP01` ACME 颁发器的例子，下面有更多的配置选项:

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: example-issuer
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: example-issuer-account-key
    solvers:
      - http01:
          ingress:
            class: nginx
```

## 选项

HTTP01 发行者支持许多附加选项。有关可用选项范围的完整详细信息，请阅读[参考文档](../../../reference/api-docs.md#acme.cert-manager.io/v1.ACMEChallengeSolverHTTP01).

### `class`

如果指定了`class`字段，cert-manager 将创建新的`Ingress`源，以便将流量路由到`acmesolver` pod，这些 pod 负责响应 ACME 挑战验证请求。

如果没有指定该字段，并且`name`也没有指定，cert-manager 将默认创建 _新_ 的`Ingress`源，但 **不会** 在这些源上设置入口类，这意味着集群中安装的 _所有_ 入口控制器将为挑战求解器提供流量，可能会产生额外的成本。

### `name`

如果指定了`name`字段，cert-manager 将编辑命名的入口源以解决 HTTP01 挑战。

这对于兼容入口控制器很有用，比如`ingress-gce`，它为每个创建的`Ingress`源使用唯一的 IP 地址。

当使用为所有入口源公开单个 IP 的入口控制器时，应该避免这种模式，因为它会与某些入口控制器特定的注释产生兼容性问题。

### `serviceType`

在极少数情况下，可能不可能/不希望使用`NodePort`作为 HTTP01 挑战响应服务的类型，例如，由于 Kubernetes 的 limit 限制。
要定义在挑战响应期间使用哪种 Kubernetes 服务类型，请指定以下 HTTP01 配置:

```yaml
http01:
  ingress:
    # Valid values are ClusterIP and NodePort
    serviceType: ClusterIP
```

默认情况下，当您不设置 HTTP01 或将`serviceType`设置为空字符串时，将使用`NodePort`类型。
通常不需要改变这个。

### `podTemplate`

您可能希望更改或添加解算器 Pod 的标签和注释。
这些可以在`podTemplate`下的`metadata`字段下配置。

类似地，你可以通过在`podTemplate`的`spec`字段下配置来设置`nodeSelector`，公差和求解器 pods 的亲和性。
不能编辑其他规格字段。

如何配置模板的示例如下:

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: ...
spec:
  acme:
    server: ...
    privateKeySecretRef:
      name: ...
    solvers:
      - http01:
          ingress:
            podTemplate:
              metadata:
                labels:
                  foo: "bar"
                  env: "prod"
              spec:
                nodeSelector:
                  bar: baz
```

添加的标签和注释将合并到 cert-manager 默认值之上，覆盖具有相同键的条目。

`podTemplate` 中不存在其他字段。

### `ingressTemplate`

可以向求解器入口源添加标签和注释。
当你在整个集群中管理多个入口控制器，并且你想要确保正确的一个将拾取并暴露解算器(用于即将解决的挑战)时，它非常有用。
这些可以在`ingressTemplate`下的`metadata`字段下配置:

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: ...
spec:
  acme:
    server: ...
    privateKeySecretRef:
      name: ...
    solvers:
      - http01:
          ingress:
            ingressTemplate:
              metadata:
                labels:
                  foo: "bar"
                annotations:
                  "nginx.ingress.kubernetes.io/whitelist-source-range": "0.0.0.0/0,::/0"
                  "nginx.org/mergeable-ingress-type": "minion"
                  "traefik.ingress.kubernetes.io/frontend-entry-points": "http"
```

添加的标签和注释将合并到 cert-manager 默认值之上，覆盖具有相同键的条目。

入口的其他字段都不能被编辑。

## 配置 HTTP-01 网关 API 解析器

**功能状态**: cert-manager 1.5 [alpha]

Gateway 和 HTTPRoute 源是[Gateway API][gwapi]的一部分，这是一组可以安装在 Kubernetes 集群上的 CRDs，它提供了对 Ingress API 的各种改进。

[gwapi]: https://gateway-api.sigs.k8s.io

!!! tip "📌 该特性需要安装[Gateway API 包](https://gateway-api.sigs.k8s.io/guides/#installing-a-gateway-controller)，并将特性标志传递给 cert-manager 控制器。"

    安装v1.5.1网关API包(网关crd和webhook)，执行以下命令:

    ```sh
    kubectl apply -f "https://github.com/kubernetes-sigs/gateway-api/releases/download/v0.5.1/standard-install.yaml"
    ```

    要在cert-manager中启用该功能，请打开`GatewayAPI`功能门:

    - 如果你使用Helm:

      ```sh
      helm upgrade --install cert-manager jetstack/cert-manager --namespace cert-manager \
        --set "extraArgs={--feature-gates=ExperimentalGatewayAPISupport=true}"
      ```

    - 如果您正在使用原始的cert-manager清单，请在cert-manager控制器部署中添加以下标志:

      ```yaml
      args:
        - --feature-gates=ExperimentalGatewayAPISupport=true
      ```

    网关API CRDs应该在启动cert-manager之前安装，或者在安装网关API crd之后重新启动cert-manager部署。
    这很重要，因为一些cert-manager组件只在启动时执行Gateway API检查。可以使用如下命令重启cert-manager。

    ```sh
    kubectl rollout restart deployment cert-manager -n cert-manager
    ```

!!! info

    🚧 cert-manager 1.8+使用v1alpha2 Kubernetes Gateway API进行测试。
    由于源转换，它也可以与v1beta1一起工作，但还没有使用它进行测试。

HTTP-01 求解器使用给定的标签创建一个临时的 HTTPRoute。
这些标签必须与在端口 80 上包含侦听器的 Gateway 匹配。

下面是一个使用网关 API 的 HTTP-01 ACME 发布者的例子:

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt
  namespace: default
spec:
  acme:
    solvers:
      - http01:
          gatewayHTTPRoute:
            parentRefs:
              - name: traefik
                namespace: traefik
                kind: Gateway
```

颁发者依赖于集群上现有的网关。cert-manager 不编辑网关源。

例如，以下网关将允许发行者解决挑战:

```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: Gateway
metadata:
  name: traefik
  namespace: traefik
spec:
  gatewayClassName: traefik
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
```

在上面的例子中，网关是专门为解决 HTTP-01 挑战而创建的，但是您也可以选择重用现有的网关，只要它在端口 80 上有一个侦听器。

只要网关的端口 80 监听器配置为`from: All`，颁发者上的“标签”可以引用位于单独名称空间上的网关。
请注意，证书仍将在与颁发者相同的名称空间上创建，这意味着您将无法在上述网关中引用此 Secret。

当上面的颁发者获得证书时，证书管理器会创建临时的 HTTPRoute。例如，使用以下证书:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-tls
  namespace: default
spec:
  issuerRef:
    name: letsencrypt
  dnsNames:
    - example.net
```

你会看到一个 HTTPRoute 出现:

```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: HTTPRoute
metadata:
  name: cm-acme-http-solver-gdhvg
  namespace: default
spec:
  parentRefs:
    - name: traefik
      namespace: traefik
      kind: Gateway
  hostnames:
    - example.net
  rules:
    - forwardTo:
        - port: 8089
          serviceName: cm-acme-http-solver-gdhvg
          weight: 1
      matches:
        - path:
            type: Exact
            value: /.well-known/acme-challenge/YadC4gaAzqEPU1Yea0D2MrzvNRWiBCtUizCtpiRQZqI
```

证书颁发后，HTTPRoute 将被删除。

<h3 id="gatewayhttproute-labels"></h3>

### `labels`

这些标签被复制到证书管理器为解决 HTTP-01 挑战而创建的临时 HTTPRoute 中。这些标签必须与集群上的一个 Gateway 源相匹配。匹配的 Gateway 在端口 80 上有一个监听器。

请注意，当标签与集群上的任何 Gateway 不匹配时，cert-manager 将创建临时 HTTPRoute 挑战，并且不会发生任何事情。

<h3 id="gatewayhttproute-service-type"></h3>

### `serviceType`

此字段与 [`http01.ingress.serviceType`](#ingress-service-type)含义相同。.

## 为 HTTP-01 求解器传播检查设置名称服务器

在尝试 HTT01 挑战之前，cert-manager 将执行可达性测试。
默认情况下，cert-manager 将使用从`/etc/resolv.conf`中获得的递归名称服务器来查询挑战 URL。

如果这不是所希望的(例如，对于分割地平线的 DNS)， cert-manager 控制器将暴露一个标志，允许您更改此行为:

`--acme-http01-solver-nameservers` cert-manager 应该查询的递归名称服务器的主机和端口的逗号分隔字符串。

使用示例:

```bash
--acme-http01-solver-nameservers="8.8.8.8:53,1.1.1.1:53"
```

如果你正在使用`cert-manager` helm chart，你可以通过`ValuesextraArgs`或在 helm 安装/升级时使用`--set`命令设置递归名称服务器:

```bash
--set 'extraArgs={--acme-http01-solver-nameservers=8.8.8.8:53\,1.1.1.1:53}'
```
