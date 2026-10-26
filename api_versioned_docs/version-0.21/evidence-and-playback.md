---
title: Evidence artifacts and Rego playback
description: Submit evidence, preserve its policy inputs, and reproduce evaluations for debugging.
---

# Evidence artifacts and Rego playback

This guide is for developers integrating with the Continuous Compliance Framework (CCF) API. Follow an evidence submission through stored policy artifacts to a reproducible Rego evaluation, with behavior verified at API `v0.21.1`.

## How it works

An evidence stream has a stable `uuid`: reuse it for the same observation of the same subject. Each submission creates a separate record with its own `id`. Retrieve or replay that record by `id`.

Artifacts preserve the policy bundle, collected input, and optional policy data. The API canonicalizes each upload, computes its SHA-256 digest, and stores identical content once. Evidence refers to those digests through `policy-artifacts`. The API checks their existence and media types and writes reserved props before signing:

| Evidence prop | Artifact |
|---|---|
| `_policy_bundle_digest` | Policy bundle |
| `_policy_input_digest` | Collected JSON input |
| `_policy_data_digest` | Optional JSON policy data |

Do not supply these props yourself, even without `policy-artifacts`: the API rejects them. Evidence without artifacts is still accepted.

Inline evaluation (`POST /api/playback/evaluate`) runs supplied source and input without storing anything. Stored playback (`GET /api/evidence/{id}/playback`) loads immutable artifacts and compares the replay with the recorded result. Both use the same evaluator as the CCF Agent.

## Before you start

Use an API with its database migrated and JWT keys configured. You need `curl`, `jq`, and `tar` for these examples. Set `API` to your API origin, without `/api`, and supply tokens through your shell environment:

```sh
export API=https://api.example.test
```

Obtain `USER_TOKEN` through `POST /api/auth/token` and `AGENT_TOKEN` through `POST /api/auth/agent/token`. See [Authentication and permissions](./authentication-and-permissions.md) and [Agent management](./agent-management.md) for credentials and enrollment. The tokens have different uses:

| Operation | Authentication and authorization |
|---|---|
| Submit evidence | User or agent token; `evidence:create`. Anonymous calls are possible when public agent endpoints are allowed. |
| Read evidence | `evidence:read`; optional user or agent token when public endpoints are allowed. |
| Upload artifacts | Agent token; `artifact:ingest`. Anonymous uploads are possible when public agent endpoints are allowed. A user token does not replace an agent token here. |
| Download artifacts or replay stored evidence | User or agent token, even when public endpoints are allowed; `artifact:read`. |
| Evaluate inline | User or agent token; `playback:execute`. Anonymous calls are possible when public agent endpoints are allowed. |
| List subjects, inspect signatures, or verify signatures | User token; `evidence:read`. |

Authentication does not grant authorization. Check your selected authorization driver's policy; all bundled Cedar roles include `playback:execute`, but other permissions vary by role.

For agent evidence, including anonymous ingest, `CCF_EVIDENCE_REQUIRE_SUBJECT` applies: `off` accepts unattributed evidence, `warn` logs it, and `enforce` rejects it. Labels matching registered component subject templates derive attributed subjects. An existing subject explicitly named by `subjects[].subject-uuid` also satisfies the check. Legacy `Identifier`/`Type` subjects alone do not.

For user submissions, `CCF_MANUAL_EVIDENCE_REQUIRE_SUBJECT=true` requires an attributed subject. Query `GET /api/evidence/config` for `data.manual-subject-required`. Pick an existing defined component, SSP system component, party, or user from `GET /api/subjects`; unknown UUIDs return `400`. The examples name an existing subject explicitly, so they also work with subject enforcement enabled.

## Enable it

Artifacts and playback are enabled by default. Set these environment variables before starting the API; restart after changes. See the [configuration reference](./configuration.md) for the full configuration.

| Setting | Purpose | Type | Default | Example |
|---|---|---|---|---|
| `CCF_ARTIFACT_MAX_BYTES` | Maximum upload body and expanded bundle size, in bytes. *(new in 0.21)* | `int` | `16777216` | `CCF_ARTIFACT_MAX_BYTES=8388608` |
| `CCF_ARTIFACT_MAX_CONCURRENT` | Uploads canonicalized and stored at once across callers. *(new in 0.21)* | `int` | `8` | `CCF_ARTIFACT_MAX_CONCURRENT=4` |
| `CCF_EVIDENCE_DEFAULT_EXPIRY_MONTHS` | Calendar months added to the evidence end time when expiry is omitted. | `int` | `1` | `CCF_EVIDENCE_DEFAULT_EXPIRY_MONTHS=3` |
| `CCF_EVIDENCE_REQUIRE_SUBJECT` | Agent subject enforcement: `off`, `warn`, or `enforce`. *(new in 0.21)* | `string` | `off` | `CCF_EVIDENCE_REQUIRE_SUBJECT=enforce` |
| `CCF_MANUAL_EVIDENCE_REQUIRE_SUBJECT` | Require attributed subjects on user evidence. *(new in 0.21)* | `bool` | `false` | `CCF_MANUAL_EVIDENCE_REQUIRE_SUBJECT=true` |
| `CCF_PLAYBACK_ENABLED` | Register both inline and stored playback routes. *(new in 0.21)* | `bool` | `true` | `CCF_PLAYBACK_ENABLED=true` |
| `CCF_PLAYBACK_MAX_BYTES` | Inline request body limit, in bytes. *(new in 0.21)* | `int` | `2097152` | `CCF_PLAYBACK_MAX_BYTES=1048576` |
| `CCF_PLAYBACK_MAX_CONCURRENT` | Shared limit for inline and stored evaluations across callers. *(new in 0.21)* | `int` | `4` | `CCF_PLAYBACK_MAX_CONCURRENT=2` |
| `CCF_PLAYBACK_TIMEOUT` | Evaluation time limit, as a Go duration. *(new in 0.21)* | `duration` | `5s` | `CCF_PLAYBACK_TIMEOUT=2s` |
| `CCF_STRICT_DISABLE_PUBLIC_AGENT_ENDPOINTS` | Require tokens on public ingest, evidence reads, and inline playback. | `bool` | `false` | `CCF_STRICT_DISABLE_PUBLIC_AGENT_ENDPOINTS=true` |

Nonpositive artifact limits, playback limits, and playback timeout fall back to defaults. Setting `CCF_PLAYBACK_ENABLED=false` removes both playback routes; artifact upload and download remain available.

The sandbox removes `http.send`, `net.lookup_ip_addr`, and `opa.runtime`; using them produces a compile error. Modules come from the request or stored bundle, never arbitrary files on the API host. Evaluation stops at the configured timeout. Preserve external observations in input instead of fetching them from Rego during playback.

## Use it

### Submit and retrieve evidence

List subjects with your user token and choose the UUID of the fictional test subject you intend to assess:

```sh
curl --fail-with-body -sS "$API/api/subjects" -H "Authorization: Bearer $USER_TOKEN"
```

Create `evidence.json`, replacing the subject placeholder with that UUID. The dates illustrate a completed observation; choose dates appropriate to your test. `start` and `end` must not be in the future.

```json
{
  "uuid": "832cbfa8-99b1-4a10-bfe1-94110d604ec9",
  "title": "Root login is disabled",
  "labels": {"environment": "fictional-lab", "_policy": "compliance_framework.ssh_root_login"},
  "subjects": [{"subject-uuid": "REPLACE_WITH_EXISTING_SUBJECT_UUID"}],
  "start": "2026-01-01T12:00:00Z",
  "end": "2026-01-01T12:00:01Z",
  "expires": "2026-02-01T12:00:01Z",
  "status": {"state": "not-satisfied"},
  "props": [{"name": "_violation_id", "value": "root-login-enabled"}]
}
```

Submit with a user token for manual evidence, or replace it with your agent token for agent evidence:

```sh
curl --fail-with-body -sS "$API/api/evidence" -H "Authorization: Bearer $USER_TOKEN" -H 'Content-Type: application/json' --data-binary @evidence.json > created.json
```

The response is `201` with a `data` object. Capture its record `id`, rather than the stream `uuid`:

```sh
EVIDENCE_ID=$(jq -er '.data.id' created.json)
```

```sh
curl --fail-with-body -sS "$API/api/evidence/$EVIDENCE_ID" -H "Authorization: Bearer $USER_TOKEN"
```

Omitting `expires`, setting it to `null`, or supplying the Unix epoch selects the default expiry: end time plus `CCF_EVIDENCE_DEFAULT_EXPIRY_MONTHS` calendar months. A zero end time uses the current time as that base. Expiry describes evidence freshness; signature verification separately checks token validity at signing time.

Authenticated submissions are signed by the API with RS256 over a canonical content hash, with a snapshot of the submitter and token claims. Anonymous public submissions have no signer and remain unsigned. The create response includes the signature when present; ordinary retrieval omits it. Inspect it with a user token:

```sh
curl --fail-with-body -sS "$API/api/evidence/$EVIDENCE_ID/signature" -H "Authorization: Bearer $USER_TOKEN"
```

```sh
curl --fail-with-body -sS -X POST "$API/api/evidence/$EVIDENCE_ID/verify" -H "Authorization: Bearer $USER_TOKEN"
```

For signed evidence, inspect `data.is_valid`, `data.checks`, and `data.errors`. Verification checks the current content hash, JWS and envelope consistency, and whether signing occurred within the submitter token's issued/not-before/expiry times. It does not require that original token to remain unexpired today. Unsigned evidence returns `status: "unsigned"` with `is_valid: false`. Keep the API's signing keys stable across restarts so historical signatures can still verify.

### Configure subject templates

Subject templates turn matching evidence labels into attributed subjects. The released evidence path materializes `component` templates with `source-mode: runtime-derived`; other template types are accepted for storage but do not produce evidence subjects. Every selector label and identity key must match/be present. Identity keys must also occur in `label-schema`. Stable identity labels let repeated observations reuse the same derived component; changing them can create a different subject.

Operators with `admin:manage` can list, create, read and update `/api/admin/subject-templates`. For example, create a template for fictional lab evidence:

```sh
curl --fail-with-body -sS "$API/api/admin/subject-templates" \
  -H "Authorization: Bearer $USER_TOKEN" -H 'Content-Type: application/json' \
  --data '{"name":"Fictional lab service","type":"component","source-mode":"runtime-derived","component-type":"service","title-template":"Fictional lab service","identity-label-keys":["asset_id"],"selector-labels":[{"key":"_plugin","value":"fictional-lab"}],"label-schema":[{"key":"asset_id"}]}'
```

Success returns `201` with `data`. Submit evidence with labels `_plugin: fictional-lab` and `asset_id: fictional-service-1`, then inspect its attributed subjects through the evidence and subject retrieval routes above. This does not require inline playback or an uploaded policy artifact. An unmatched template or missing identity label does not derive a subject; strict enforcement can therefore reject the evidence.

CCF Agents reconcile their complete template set with `POST /api/agent/subject-templates/batch`, using `subject-template:update` and the agent ingestion authentication policy. Its JSON body contains `plugin-id` and a required `templates` array. Templates are scoped by selector label `_plugin`; each batch item must include `_plugin` equal to the batch `plugin-id`. **Omitted templates are deleted** within that scope, and `templates: []` deletes the entire scoped set. Missing/null `templates` returns `400`. Success returns `200` with `data.created`, `updated`, `deleted`, `unchanged` and `warnings`; non-component templates produce warnings. Preserve the full desired set when reconciling. Use this version's **API reference**, under Subject Templates, for batch item fields and limits.

### Upload and attach artifacts

Create `policy.rego` with the policy below. It uses Rego v1 syntax:

```rego
package compliance_framework.ssh_root_login

title := "Root login is disabled"

violation contains {"id": "root-login-enabled"} if {
    input.sshd_config.PermitRootLogin == data.forbidden
}
```

Package it as an OPA bundle tar:

```sh
tar -cf bundle.tar policy.rego
```

Create `input.json`:

```json
{"sshd_config":{"PermitRootLogin":"yes"}}
```

Create `data.json`:

```json
{"forbidden":"yes"}
```

Upload the three artifacts and capture each returned digest. Responses contain `{digest, mediaType, sizeBytes}` directly, without a `data` wrapper. New content returns `201`; an existing artifact returns `200`.

```sh
BUNDLE_DIGEST=$(curl --fail-with-body -sS "$API/api/agent/artifacts" -H "Authorization: Bearer $AGENT_TOKEN" -H 'Content-Type: application/vnd.ccf.policy-bundle.v1+tar' --data-binary @bundle.tar | jq -er '.digest')
```

```sh
INPUT_DIGEST=$(curl --fail-with-body -sS "$API/api/agent/artifacts" -H "Authorization: Bearer $AGENT_TOKEN" -H 'Content-Type: application/json' --data-binary @input.json | jq -er '.digest')
```

```sh
DATA_DIGEST=$(curl --fail-with-body -sS "$API/api/agent/artifacts" -H "Authorization: Bearer $AGENT_TOKEN" -H 'Content-Type: application/json' --data-binary @data.json | jq -er '.digest')
```

JSON canonicalization sorts object keys, removes insignificant whitespace, avoids HTML escaping, and preserves number spellings. Bundles accept tar or gzipped tar. The API stores an uncompressed tar with sorted regular files, mode `0644`, owner `0/0`, and zero modification times. It rejects links, duplicate or escaping paths, and bundles OPA cannot read. Use the returned `sha256:` digest followed by 64 lowercase hex characters; do not hash the original upload yourself.

Add the returned references to the same evidence payload:

```sh
jq --arg bundle "$BUNDLE_DIGEST" --arg input "$INPUT_DIGEST" --arg data "$DATA_DIGEST" '. + {"policy-artifacts":{"bundle-digest":$bundle,"input-digest":$input,"policy-data-digest":$data}}' evidence.json > evidence-with-artifacts.json
```

```sh
curl --fail-with-body -sS "$API/api/evidence" -H "Authorization: Bearer $AGENT_TOKEN" -H 'Content-Type: application/json' --data-binary @evidence-with-artifacts.json > created.json
```

```sh
EVIDENCE_ID=$(jq -er '.data.id' created.json)
```

Policy data is optional: omit `policy-data-digest` when no separate data is configured. This example needs it because the rule reads `data.forbidden`. Stored playback layers separate policy data over the bundle's data documents; supply a JSON object for that overlay, even though artifact uploads accept any JSON value.

Download canonical input by digest; the same route serves bundle and policy data digests with their original media types:

```sh
curl --fail-with-body -sS "$API/api/artifacts/$INPUT_DIGEST" -H "Authorization: Bearer $USER_TOKEN" -o canonical-input.json
```

### Replay the stored record

```sh
curl --fail-with-body -sS "$API/api/evidence/$EVIDENCE_ID/playback" -H "Authorization: Bearer $USER_TOKEN"
```

The response wraps the playback in `data`. Expect `available: true`, `replay.status: "not-satisfied"`, and `comparison.statusMatches: true`, with empty `missingViolationIds` and `newViolationIds` for this example. Playback pins `time.now_ns()` to the evidence's `end`. The `_policy` label selects the package; without it, selection works only when the bundle yields one result.

The response includes policy files, pretty-printed JSON strings, recorded and replayed results, and comparison details. Violation `rules` identify stored file paths and 1-based line ranges where possible; location failures leave an empty list. `unidentifiedViolations` counts replayed violations without IDs. Displayed input is truncated at 1 MiB with `inputTruncated: true`; download its artifact for the full input.

Older evidence without artifacts, or evidence whose artifacts are missing, returns `200` with `data.available: false` and a `reason`. It cannot reconstruct an input that was never stored. Sandbox failures return the artifacts and `errors`, with no `replay`; HTTP success alone does not mean the replay succeeded.

### Evaluate inline with a pinned time

Send this complete request with `POST /api/playback/evaluate`. It is independent of the stored record and reproduces the same violation:

```sh
curl --fail-with-body -sS "$API/api/playback/evaluate" -H "Authorization: Bearer $USER_TOKEN" -H 'Content-Type: application/json' --data-binary @- <<'JSON'
{
  "policy": "package compliance_framework.ssh_root_login\ntitle := \"Root login is disabled\"\nviolation contains {\"id\": \"root-login-enabled\"} if { input.sshd_config.PermitRootLogin == data.forbidden }\n",
  "input": {"sshd_config":{"PermitRootLogin":"yes"}},
  "data": {"forbidden":"yes"},
  "evaluatedAt": "2026-01-01T12:00:01Z"
}
JSON
```

The response has no `data` wrapper. Its `results` includes package `compliance_framework.ssh_root_login`, title `Root login is disabled`, status `not-satisfied`, and `violations: [{"id":"root-login-enabled"}]`. `prints` is empty and `durationMs` varies. Change the input to `"no"` to get `satisfied` with no violations. A nonempty `skip_reason` produces `skipped`.

`evaluatedAt` is RFC 3339 and controls `time.now_ns()`; omit it to evaluate at the current time. Optional `modules` maps filenames to helper Rego source. Packages outside `compliance_framework` are available for imports but are not evaluated as evidence. Inspect each result's `error` as well as its status: a missing evidence title, for example, prevents recording that result as evidence. Inline evaluation neither stores artifacts nor signs or submits evidence.

## Troubleshooting

| Symptom | What to check or change |
|---|---|
| `400` submitting evidence | Fix malformed JSON, future start/end times, unknown subjects, or subject enforcement. For artifact references, upload first, use returned digests of the right media type, and remove reserved digest props. |
| `400` uploading an artifact | Check `Content-Type`, JSON validity, tar paths, and OPA bundle readability. Expanded bundles over `CCF_ARTIFACT_MAX_BYTES` fail canonicalization with `400`. |
| `400` evaluating inline | Supply nonempty `policy` and non-null `input`, valid JSON, and a valid RFC 3339 `evaluatedAt`. |
| `401` or `403` | Refresh the correct token kind and check authorization. Strict mode denies anonymous calls; downloads and stored playback always require a token. Signature inspection and verification require a user token. |
| `404` on playback | Check the record `id` and whether `CCF_PLAYBACK_ENABLED` is `false`. For artifact downloads, check that the digest is stored. |
| `413` | Reduce the upload body below `CCF_ARTIFACT_MAX_BYTES`, or the inline body below `CCF_PLAYBACK_MAX_BYTES`. Review limits before raising them. |
| `422` evaluating inline | Inspect `errors[].code`, `message`, `file`, `row`, and `col`. Fix parse/type/conflict errors; remove forbidden builtins; add a `compliance_framework` package for `no_policies`; simplify the rule or adjust timeout for `eval_timeout`. |
| `429` | Retry with backoff. Uploads wait up to 10 seconds for a slot after reading the body. Playback rejects immediately when the shared evaluation slots are full. Reduce parallel requests before raising concurrency. |
| Stored playback differs | Check `_policy`, evidence end time, input, data overlay, recorded status, and `_violation_id` props. A signature validates content integrity, not that the recorded outcome matches its policy. |
| Signature verification fails | Inspect `checks` and `errors`; confirm persistent signing keys, unchanged signed content, and valid token times at signing. |

## Related

- [API overview](./index.md)
- [Configuration reference](./configuration.md)
- [Authentication and permissions](./authentication-and-permissions.md)
- [Agent management](./agent-management.md)
- The **API reference** in this version's sidebar provides the complete request and response schemas.
