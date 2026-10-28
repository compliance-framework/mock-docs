---
title: AWS CloudFront policy input
description: The distribution object passed to AWS CloudFront Rego policies in plugin-aws-cloudfront v0.1.0.
sidebar_position: 2
---

# AWS CloudFront policy input

Use this reference when you write AWS CloudFront policies for the Continuous Compliance Framework (CCF). It describes the input collected by `plugin-aws-cloudfront` [v0.1.0](https://github.com/compliance-framework/plugin-aws-cloudfront/releases/tag/v0.1.0), in the `0.1` plugin line.

## How collection works

On each evaluation requested by the CCF Agent, the plugin calls STS `GetCallerIdentity` to obtain the account ID. It paginates CloudFront `ListDistributions` and calls `ListTagsForResource` for each distribution. CloudFront is global; the plugin routes these API calls through `us-east-1`. The CCF Agent determines when evaluations run; the plugin does not set a collection interval.

The plugin evaluates each configured policy path for **one distribution at a time**. Your Rego policy receives a flat object at `input`. Read the distribution ID as `input.distribution_id`.

The input comes from [`DistributionContext` and `ToOPAInput()`](https://github.com/compliance-framework/plugin-aws-cloudfront/blob/v0.1.0/internal/data.go). The [policy evaluator](https://github.com/compliance-framework/plugin-aws-cloudfront/blob/v0.1.0/internal/eval.go) passes this object to the policy processor. Bundle `data.json` and configured policy data populate the separate Rego `data` document. Configured policy data overrides bundle data at the top level. The `policy_labels` setting adds evidence labels; it does not add fields to `input`.

## Input shape

This example uses fictional identifiers and a reserved example domain. It includes the optional certificate ARN, the default cache behavior, and one path-based cache behavior.

```json
{
  "account_id": "000000000000",
  "distribution_id": "EDEMO123EXAMPLE",
  "arn": "arn:aws:cloudfront::000000000000:distribution/EDEMO123EXAMPLE",
  "domain_name": "fictional-distribution.example.com",
  "status": "Deployed",
  "enabled": true,
  "certificate_arn": "arn:aws:acm:us-east-1:000000000000:certificate/00000000-0000-4000-8000-000000000001",
  "cache_behaviours": [
    {
      "path_pattern": "/*",
      "viewer_protocol_policy": "redirect-to-https"
    },
    {
      "path_pattern": "/api/*",
      "viewer_protocol_policy": "https-only"
    }
  ],
  "tags": {
    "environment": "demo"
  }
}
```

### Fields

Keys and values are case-sensitive. Use the JSON key `cache_behaviours`, including its spelling, when you write policies.

| Field | JSON type | Presence | Meaning |
|---|---|---|---|
| `account_id` | string | Always | AWS account ID returned by STS `GetCallerIdentity`. |
| `distribution_id` | string | Always | CloudFront distribution ID. |
| `arn` | string | Always | CloudFront distribution ARN. |
| `domain_name` | string | Always | Distribution domain name returned by CloudFront. |
| `status` | string | Always | Distribution deployment status, such as `Deployed` or `InProgress`. |
| `enabled` | boolean | Always | Whether the distribution is enabled. |
| `certificate_arn` | string | Optional | `ViewerCertificate.ACMCertificateArn`; omitted when empty or unavailable. |
| `cache_behaviours` | array of objects | Always | Default cache behavior first, when present, followed by path-based cache behaviors in the order returned by CloudFront. |
| `tags` | object of string values | Always | Distribution tags keyed by tag name, such as `{"environment": "demo"}`. |

Each object in `cache_behaviours` contains these two fields:

| Field | JSON type | Meaning |
|---|---|---|
| `path_pattern` | string | The plugin assigns `/*` to the default cache behavior. Other entries use the path pattern returned by CloudFront. |
| `viewer_protocol_policy` | string | CloudFront viewer protocol policy: `allow-all`, `redirect-to-https`, or `https-only`. |

### Missing and empty values

Guard against a missing `certificate_arn`. The plugin only copies an ACM certificate ARN; it does not include an IAM certificate ID or a flag for the default CloudFront certificate.

Other string fields remain present and can be `""` when AWS provides no value. An unavailable `enabled` value becomes `false`. For distributions returned by the collector, empty `cache_behaviours` is `[]` and empty `tags` is `{}`. A tag lookup failure also produces `{}`, so empty tags do not prove that the distribution has no tags. A zero-value `DistributionContext` constructed outside the collector can serialize these collections as `null`.

The input contains only the fields listed above. It does not include a `region`, origins, aliases, a minimum TLS version, or the full CloudFront response.

## Read the input in Rego

This helper identifies a cache behavior that allows HTTP. Add it to your policy bundle's rules as needed; it is not a complete CCF policy bundle.

```rego
package fictional.cloudfront

import rego.v1

default allows_http := false

allows_http if {
    some behavior in input.cache_behaviours
    behavior.viewer_protocol_policy == "allow-all"
}
```

For the JSON example above, `allows_http` is `false`. Changing either behavior's `viewer_protocol_policy` to `allow-all` makes it `true`. An empty array also produces `false`; check for missing cache behaviors separately if your policy requires them.

## Related

- [AWS CloudFront plugin and policy bundles](./index.md)
- [Collector and input serialization source at v0.1.0](https://github.com/compliance-framework/plugin-aws-cloudfront/blob/v0.1.0/internal/data.go)
- [Per-distribution policy evaluation source at v0.1.0](https://github.com/compliance-framework/plugin-aws-cloudfront/blob/v0.1.0/internal/eval.go)
