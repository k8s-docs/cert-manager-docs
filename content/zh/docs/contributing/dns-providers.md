---
title: "DNS Providers"
linkTitle: "DNS 供应商"
weight: 70
type: "docs"
---

## Creating DNS Providers

Due to the large number of requests to support DNS providers to resolve DNS
challenges, it have become unpractical and unfeasible to maintain and test all
coming in. For this reason, it has been decided to instead support out-of-tree
DNS providers via way of an external webhook.

To implement an external DNS provider webhook, it is recommended to base your
implementation on the [example
repository](https://github.com/jetstack/cert-manager-webhook-example). Please
reach out on the `cert-manager-dev` channel on the [community
slack](https://slack.k8s.io) for advise and guidance on getting a DNS webhook
running and released.
