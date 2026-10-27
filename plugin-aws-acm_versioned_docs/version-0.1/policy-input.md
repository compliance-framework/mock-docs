---
title: AWS ACM policy input
description: The certificate object passed to AWS ACM Rego policies in plugin-aws-acm v0.1.1.
sidebar_position: 2
---

# AWS ACM policy input

Use this reference when you write custom AWS ACM policies for the Continuous Compliance Framework (CCF). It describes the input collected by `plugin-aws-acm` [v0.1.1](https://github.com/compliance-framework/plugin-aws-acm/releases/tag/v0.1.1), in the `0.1` plugin line.

## How collection works

On each evaluation requested by the CCF Agent, the plugin lists certificates in every configured region. For each certificate, it calls `DescribeCertificate` and `ListTagsForCertificate`. The CCF Agent configuration determines when evaluations run; the plugin does not set a collection interval.

The plugin evaluates each configured policy path separately for **one certificate at a time**. Your Rego policy receives a flat object at `input`, not an array or an object wrapped in `certificate` or `certificates`. Read the certificate ARN as `input.certificate_arn`.

The input comes from [`CertificateContext` and `ToOPAInput()`](https://github.com/compliance-framework/plugin-aws-acm/blob/v0.1.1/internal/data.go). The [policy evaluator](https://github.com/compliance-framework/plugin-aws-acm/blob/v0.1.1/internal/eval.go) passes that object to the policy processor. Policy data supplied during configuration is passed separately to the processor, and `policy_labels` adds evidence labels; neither is a field in this certificate object.

## Input shape

This example uses fictional certificate and resource identifiers. It includes all optional fields.

```json
{
  "region": "us-east-1",
  "account_id": "000000000000",
  "certificate_arn": "arn:aws:acm:us-east-1:000000000000:certificate/00000000-0000-4000-8000-000000000001",
  "domain_name": "service.example.com",
  "status": "ISSUED",
  "not_after": "2027-10-01T00:00:00Z",
  "issued_at": "2026-09-01T00:00:00Z",
  "key_algorithm": "RSA_2048",
  "type": "AMAZON_ISSUED",
  "renewal_eligibility": "ELIGIBLE",
  "transparency_logging_preference": "ENABLED",
  "domain_validation_options": [
    {
      "domain_name": "service.example.com",
      "validation_method": "DNS"
    }
  ],
  "in_use_by": [
    "arn:aws:elasticloadbalancing:us-east-1:000000000000:loadbalancer/app/fictional-service/0000000000000000"
  ],
  "tags": {
    "environment": "demo"
  }
}
```

### Fields

Keys and values are case-sensitive. The plugin uses JSON field names rather than the AWS API's PascalCase names.

| Field | JSON type | Presence | Meaning |
|---|---|---|---|
| `region` | string | Always | Configured AWS region used to fetch the certificate. |
| `account_id` | string | Always | Account ID extracted from the certificate ARN. |
| `certificate_arn` | string | Always | Certificate ARN returned by `ListCertificates`. |
| `domain_name` | string | Always | Primary domain name from `DescribeCertificate`. |
| `status` | string | Always | ACM certificate status, such as `ISSUED`, `PENDING_VALIDATION`, or `EXPIRED`. |
| `not_after` | string | Optional | Expiration time as an RFC 3339 timestamp, with fractional seconds when present. Omitted when ACM supplies no value. |
| `issued_at` | string | Optional | Issue time in the same timestamp format. Omitted when ACM supplies no value. |
| `key_algorithm` | string | Always | ACM key algorithm with hyphens replaced by underscores, such as `RSA_2048` or `EC_prime256v1`. |
| `type` | string | Optional | Certificate type, such as `AMAZON_ISSUED`, `IMPORTED`, or `PRIVATE`. Omitted when empty. |
| `renewal_eligibility` | string | Optional | ACM renewal eligibility, such as `ELIGIBLE` or `INELIGIBLE`. Omitted when empty. |
| `transparency_logging_preference` | string | Always | Certificate transparency logging preference from the certificate's options, such as `ENABLED` or `DISABLED`. Empty string if unavailable. |
| `domain_validation_options` | array of objects | Always | Per-domain validation details. Each object contains only `domain_name` and `validation_method`, both strings. The method is an ACM value such as `DNS` or `EMAIL`, or an empty string when unavailable. |
| `in_use_by` | array of strings | Always | ARNs of AWS resources using the certificate. |
| `tags` | object of string values | Always | Certificate tags keyed by tag name, such as `{"environment": "demo"}`. |

### Missing and empty values

The four optional fields are omitted, rather than set to `null`, when unavailable. Guard against missing timestamps before parsing them or comparing dates.

Other string fields remain present and can be `""` when ACM provides no value. For certificates returned by the collector, empty `domain_validation_options` and `in_use_by` are `[]`, and empty `tags` is `{}`. The plugin normalizes these collections during fetching. A zero-value `CertificateContext` constructed outside the collector can serialize those collections as `null`; use the collected shape for your policy fixtures.

The nested domain validation objects do not include validation status, resource records, or the full ACM response. Other ACM certificate fields are also absent unless listed above.

## Read the input in Rego

This Rego helper checks that ACM reports an issued certificate with at least one DNS validation entry. Add it to your policy bundle's rules as needed; it is not a complete CCF policy bundle.

```rego
package fictional.acm

import rego.v1

default issued_with_dns_validation := false

issued_with_dns_validation if {
    input.status == "ISSUED"
    some option in input.domain_validation_options
    option.validation_method == "DNS"
}
```

## Related

- [AWS ACM requirements, configuration, and policy bundles](./index.md)
- [Collector and input serialization source at v0.1.1](https://github.com/compliance-framework/plugin-aws-acm/blob/v0.1.1/internal/data.go)
- [Per-certificate policy evaluation source at v0.1.1](https://github.com/compliance-framework/plugin-aws-acm/blob/v0.1.1/internal/eval.go)
