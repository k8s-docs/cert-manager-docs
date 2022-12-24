# Route53

使用 Amazon AWS Route53 DNS 的 ACME DNS-01 挑战

本指南解释了如何设置`Issuer` 或 `ClusterIssuer`，以使用 Amazon Route53 解决 DNS01 ACME 挑战。
建议您先阅读[DNS01 挑战提供程序](./README.md)页面，以更全面地了解 cert-manager 如何处理 DNS01 挑战。

!!! Note

    本指南假设您的集群托管在Amazon Web Services (AWS)上，并且您已经在Route53中有一个托管区域。

## 配置 IAM 角色

cert-manager 需要能够向 Route53 添加记录，以解决 DNS01 挑战。
为此，需要创建具有以下权限的 IAM 策略:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "route53:GetChange",
      "Resource": "arn:aws:route53:::change/*"
    },
    {
      "Effect": "Allow",
      "Action": ["route53:ChangeResourceRecordSets", "route53:ListResourceRecordSets"],
      "Resource": "arn:aws:route53:::hostedzone/*"
    },
    {
      "Effect": "Allow",
      "Action": "route53:ListHostedZonesByName",
      "Resource": "*"
    }
  ]
}
```

!!! Note

    The `route53:ListHostedZonesByName` statement can be removed if you  specify the (optional) `hostedZoneID`. You can further tighten the policy by limiting the hosted zone that cert-manager has access to (e.g. `arn:aws:route53:::hostedzone/DIKER8JEXAMPLE`).

## 凭证

您有两个设置选项—创建用户或角色并从上面附加该策略。
使用角色被认为是最佳实践，因为您不必在秘密中存储永久凭据。

cert-manager supports two ways of specifying credentials:

- explicit by providing a `accessKeyID` and `secretAccessKey`
- or implicit (using [metadata
  service](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html)
  or [environment variables or credentials
  file](https://docs.aws.amazon.com/sdk-for-go/v1/developer-guide/configuring-sdk.html#specifying-credentials).

cert-manager also supports specifying a `role` to enable cross-account access
and/or limit the access of cert-manager. Integration with
[`kiam`](https://github.com/uswitch/kiam) and
[`kube2iam`](https://github.com/jtblin/kube2iam) should work out of the box.

## 跨帐户访问

!!! Example "Account Y manages Route53 DNS Zones. "

    现在，您希望在帐户X(或许多其他帐户)中运行的cert-manager能够管理帐户Y中托管的Route53区域中的记录。

First, create a role with the permissions policy above (let's call the role `dns-manager`)
in Account Y, and attach a trust relationship like the one below.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::XXXXXXXXXXX:role/cert-manager"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Bear in mind, that you won't be able to define this policy until `cert-manager` role on account Y is created. If you are setting this up using a configuration language, you may want to define principal as:

```json
"Principal": {
        "AWS": "XXXXXXXXXXX"
      }
```

And restrict it, in a future step, after all the roles are created.

This allows the role `cert-manager` in Account X to assume the `dns-manager` role in Account Y to manage the Route53 DNS zones in Account Y. For more information visit the [official
documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/tutorial_cross-account-with-roles.html).

Second, create the cert-manager role in Account X; this will be used as a credentials source for the cert-manager pods running in Account X. Attach to the role the following **permissions** policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Resource": "arn:aws:iam::YYYYYYYYYYYY:role/dns-manager",
      "Action": "sts:AssumeRole"
    }
  ]
}
```

And the following trust relationship (Add AWS `Service`s as needed):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

## 创建颁发者 (or `ClusterIssuer`)

Here is an example configuration for a `ClusterIssuer`:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    ...
    solvers:

    # example: cross-account zone management for example.com
    # this solver uses ambient credentials (i.e. inferred from the environment or EC2 Metadata Service)
    # to assume a role in a different account
    - selector:
        dnsZones:
          - "example.com"
      dns01:
        route53:
          region: us-east-1
          hostedZoneID: DIKER8JEXAMPLE # optional, see policy above
          role: arn:aws:iam::YYYYYYYYYYYY:role/dns-manager

    # this solver handles example.org challenges
    # and uses explicit credentials
    - selector:
        dnsZones:
          - "example.org"
      dns01:
        route53:
          region: eu-central-1
          accessKeyID: AKIAIOSFODNN7EXAMPLE
          secretAccessKeySecretRef:
            name: prod-route53-credentials-secret
            key: secret-access-key
          # you can also assume a role with these credentials
          role: arn:aws:iam::YYYYYYYYYYYY:role/dns-manager
```

!!! Note

    as mentioned above, the pod is using `arn:aws:iam::XXXXXXXXXXX:role/cert-manager` as a credentials source in Account X, but the `ClusterIssuer` ultimately assumes the `arn:aws:iam::YYYYYYYYYYYY:role/dns-manager` role to actually make changes in Route53 zones located in Account Y.

## 服务帐户的 EKS IAM 角色(IRSA)

While [`kiam`](https://github.com/uswitch/kiam) / [`kube2iam`](https://github.com/jtblin/kube2iam) work directly with cert-manager, some special attention is needed for using the [IAM Roles for Service Accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) feature available on EKS.

### OIDC 提供者

First follow the AWS documentation [Enabling IAM roles for service accounts on your cluster](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html) to ensure that the OIDC provider for the EKS cluster is enabled. The OIDC information is needed to create the trust relationship for the cert-manager role below.

### IAM 角色信任策略

The cert-manager role needs the following trust relationship attached to the role in order to use the IRSA method. Replace the following:

- `<aws-account-id>` with the AWS account ID of the EKS cluster.
- `<aws-region>` with the region where the EKS cluster is located.
- `<eks-hash>` with the hash in the EKS API URL; this will be a random 32 character hex string (example: `45DABD88EEE3A227AF0FA468BE4EF0B5`)
- `<namespace>` with the namespace where cert-manager is running.
- `<service-account-name>` with the name of the `ServiceAccount` object created by cert-manager.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Principal": {
        "Federated": "arn:aws:iam::<aws-account-id>:oidc-provider/oidc.eks.<aws-region>.amazonaws.com/id/<eks-hash>"
      },
      "Condition": {
        "StringEquals": {
          "oidc.eks.<aws-region>.amazonaws.com/id/<eks-hash>:sub": "system:serviceaccount:<namespace>:<service-account-name>"
        }
      }
    }
  ]
}
```

!!! Note

    If you're following the Cross Account example above, this trust policy is attached to the cert-manager role in Account X with ARN `arn:aws:iam::XXXXXXXXXXX:role/cert-manager`. The permissions policy is the same as above.

### 服务注释

Annotate the `ServiceAccount` created by cert-manager:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::XXXXXXXXXXX:role/cert-manager
```

You will also need to modify the cert-manager `Deployment` with the correct file system permissions, so the `ServiceAccount` token can be read.

```yaml
spec:
  template:
    spec:
      securityContext:
        fsGroup: 1001
```

The cert-manager Helm chart provides a variable for injecting annotations into cert-manager's `ServiceAccount` and `Deployment` object like so:

```yaml
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::XXXXXXXXXXX:role/cert-manager
securityContext:
  fsGroup: 1001
```

!!! Note

    If you're following the Cross Account example above, modify the `ClusterIssuer` in the same way as above with the role from Account Y.
