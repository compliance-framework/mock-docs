---
title: Dashboards and AI filter suggestions
description: Build evidence-backed dashboards and review optional AI suggestions for SSP control mappings.
---

import Link from '@docusaurus/Link';
import useBaseUrl from '@docusaurus/useBaseUrl';

# Dashboards and AI filter suggestions

Use evidence filters to build dashboards in the Continuous Compliance Framework (CCF). This guide is for developers integrating with the API and operators enabling optional AI suggestions.

## How it works

A dashboard is a named label filter associated with controls or system components. Its expression selects evidence; compliance endpoints aggregate the matching evidence statuses. A filter can be global (`sspId` is null) or bound to one system security plan (SSP). SSP binding controls filter visibility. By default it does not add evidence-label conditions. The next-release source also supports `followSspScope=true` on create/update: it stores the base expression and combines it with the SSP evidence scope. Include the asset or environment conditions you need in the expression when using the default behavior.

Start with a manual filter. AI suggestions optionally propose evidence-to-control mappings for the controls resolved through an SSP's profiles. Generation divides controls and evidence label sets into a grid of cells and queues provider calls. Results stay pending until you review them. Generated names, mappings and confidence scores vary; they are not proof of compliance.

Filter generalization uses a deterministic detector, without a provider call. It proposes merging near-duplicate SSP filters by dropping one instance label. You review these proposals through the same decision endpoints.

## Before you start

- The integration below is verified against API `v0.21.1` and the next-release source (`dc4b3f9bfc6d1ca08fb39d639e1021f9dabc0acb`). Run database migrations and configure sign-in as described in the [API overview](./index.md).
- Have an SSP with linked profiles and resolved catalog controls. Know the SSP UUID, catalog UUID and exact stored control ID. The examples use `ac-3`, the OSCAL identifier for NIST SP 800-53 `AC-3`; substitute your catalog's identifier.
- Submit labeled evidence first; see [Evidence artifacts and Rego playback](./evidence-and-playback.md). The manual example expects evidence labeled `environment=fictional-demo`. For suggestions, label sets come from the latest evidence streams; supply meaningful SSP descriptions, components and control implementation text.
- Use a user token. With the `cedar` driver, `contributor` or `admin` can create filters, start runs and decide suggestions. `viewer` and `auditor` can read and preview. The guards require `filter:create/update` for manual writes and `dashboard-suggestion:create` for generate/generalize, `read` for preview/list/progress, and `update` for accept/reject/edit. AI diagnostics additionally require admin access. The default `builtin` driver allows signed-in users these non-admin operations; external policies may differ. See [Authentication and permissions](./authentication-and-permissions.md).
- For provider generation, enable AI and background workers, configure an Anthropic key, and allow outbound access to the provider. Manual filters need neither AI nor a provider key.

The shell examples use `curl` and `jq`. Set `API` to your deployment's URL including `/api`, `TOKEN` to your user token, and `SSP_ID` to an existing SSP UUID:

```sh
export API='https://ccf.example.test/api' TOKEN='<user-token>' SSP_ID='<ssp-uuid>'
```

## Enable it

### Create a manual dashboard first

Save this as `filter.json`, replacing `sspId` and `controls` with existing values. Use a control ID that is unambiguous across your imported catalogs: this create endpoint looks up controls by ID, not by catalog UUID. Controls and components are mutually exclusive associations.

```json
{
  "name": "Fictional demo evidence",
  "sspId": "11111111-1111-4111-8111-111111111111",
  "controls": ["ac-3"],
  "filter": {
    "scope": {
      "condition": {
        "label": "environment",
        "operator": "=",
        "value": "fictional-demo"
      }
    }
  }
}
```

```sh
curl --fail-with-body -sS -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' --data-binary @filter.json "$API/filters" > filter-response.json
```

```sh
export FILTER_ID="$(jq -r '.data.id' filter-response.json)"
```

### Enable optional provider suggestions

Set these environment variables in the API process and restart it. Inject the real key through your deployment's secret management. The example lowers queue concurrency and caps planned calls for an initial run.

```sh
export CCF_AI_ENABLED=true
export CCF_AI_PROVIDER=anthropic
export CCF_AI_API_KEY='<provider-key>'
export CCF_AI_MODEL=claude-opus-4-8
export CCF_AI_REQUEST_TIMEOUT=120s
export CCF_AI_MAX_CONTROLS_PER_CHUNK=40
export CCF_AI_MAX_LABEL_SETS_PER_CHUNK=200
export CCF_AI_QUEUE_WORKERS=2
export CCF_AI_MAX_CALLS_PER_RUN=10
export CCF_AI_MAX_SUGGESTIONS_PER_RUN=500
export CCF_WORKER_ENABLED=true
```

These names and defaults come from `cmd/root.go`, `internal/config/ai.go` and the worker configuration at `v0.21.1`:

| Setting | Purpose | Type | Default | Example |
|---|---|---|---|---|
| `CCF_AI_API_KEY` | Authenticate provider requests; required for successful generation. | `string` | Unset | `CCF_AI_API_KEY=<secret>` |
| `CCF_AI_BASE_URL` | Override the Anthropic SDK endpoint, such as for a compatible gateway. | `string` | Unset; SDK endpoint | `CCF_AI_BASE_URL=https://ai-gateway.example.test` |
| `CCF_AI_ENABLED` | Enable provider suggestions and register the suggestion worker. | `bool` | `false` | `CCF_AI_ENABLED=true` |
| `CCF_AI_MAX_CALLS_PER_RUN` | Reject runs whose planned cell count exceeds this value; `0` means unlimited planned calls. Retries can add provider requests. | `int` | `0` | `CCF_AI_MAX_CALLS_PER_RUN=10` |
| `CCF_AI_MAX_CONTROLS_PER_CHUNK` | Maximum controls per grid cell. | `int` | `40` | `CCF_AI_MAX_CONTROLS_PER_CHUNK=20` |
| `CCF_AI_MAX_LABEL_SETS_PER_CHUNK` | Maximum evidence label sets per grid cell. | `int` | `200` | `CCF_AI_MAX_LABEL_SETS_PER_CHUNK=100` |
| `CCF_AI_MAX_SUGGESTIONS_PER_RUN` | Cap inserted suggestions for a run. | `int` | `500` | `CCF_AI_MAX_SUGGESTIONS_PER_RUN=100` |
| `CCF_AI_MODEL` | Provider model identifier; use a model available to your account. | `string` | `claude-opus-4-8` | `CCF_AI_MODEL=claude-opus-4-8` |
| `CCF_AI_PROVIDER` | Supported provider: `anthropic`. Other values fail validation when enabled. | `string` | `anthropic` | `CCF_AI_PROVIDER=anthropic` |
| `CCF_AI_QUEUE_WORKERS` | Concurrent workers on the `suggestion` queue. | `int` | `4` | `CCF_AI_QUEUE_WORKERS=2` |
| `CCF_AI_REQUEST_TIMEOUT` | Timeout per provider request, in seconds or another Go duration unit. | `duration` | `120s` | `CCF_AI_REQUEST_TIMEOUT=180s` |
| `CCF_WORKER_ENABLED` | Start background workers; required to enqueue provider work. | `bool` | `true` | `CCF_WORKER_ENABLED=true` |
| `CCF_WORKER_USE_POLLING` | Poll River jobs instead of PostgreSQL LISTEN/NOTIFY. | `bool` | `false` | `CCF_WORKER_USE_POLLING=true` |

The release also reads the following internal Viper keys. Its CLI does **not** bind them to environment variables or load an AI YAML file. The stock binary uses the fallback values below; setting similarly named `CCF_AI_*` variables does not configure them. They matter when embedding the configuration loader in custom Go code.

| Setting | Purpose | Type | Default | Example |
|---|---|---|---|---|
| `ai_generalizable_label_keys` | Instance labels the deterministic detector may drop; never include meaning-bearing `_policy` or `type`. | `list` | `provider, region, account, repository, organization, environment, host, namespace, project, cluster` | `viper.Set("ai_generalizable_label_keys", []string{"region", "account"})` |
| `ai_generalization_min_shared_controls` | Minimum common controls for a generalization candidate. | `int` | `1` | `viper.Set("ai_generalization_min_shared_controls", 2)` |
| `ai_max_output_tokens` | Maximum model response tokens per cell. | `int` | `8192` | `viper.Set("ai_max_output_tokens", 8192)` |

Use positive chunk, worker, timeout and suggestion values. Zero values fall back to defaults except the planned-call limit. A cell's worker timeout is at least five minutes, or twice the request timeout plus 30 seconds if greater.

### Understand provider data

Provider requests include the SSP name and description; component titles, types, purposes and descriptions; control catalog identifiers, titles, statements and implementation text; label-key documentation; evidence label sets, evidence counts and sample titles; and requested generation constraints. The worker sends these as prompt text with structured output instructions. It does not send evidence artifact bodies through this flow. Labels and descriptions can still contain sensitive data: review what you store and which evidence you select before enabling it. Provider prompt caching is used; apply your organization's provider data-handling requirements.

## Use it

### Read dashboard results

List filters visible to your SSP. The list includes global filters and filters bound to that SSP:

```sh
curl --fail-with-body -sS -H "Authorization: Bearer $TOKEN" "$API/filters?sspId=$SSP_ID"
```

Read the manual dashboard's evidence status counts:

```sh
curl --fail-with-body -sS -H "Authorization: Bearer $TOKEN" "$API/evidence/compliance-by-filter/$FILTER_ID"
```

For control `ac-3`, aggregate through its visible filters:

```sh
curl --fail-with-body -sS -H "Authorization: Bearer $TOKEN" "$API/evidence/compliance-by-control/ac-3?sspId=$SSP_ID"
```

Only evidence matching `environment=fictional-demo` contributes through the example filter. Adding an SSP binding alone does not restrict the evidence to that SSP. An empty result can mean no matching evidence, rather than a passing control. See the <Link to={useBaseUrl('/api/0.21/reference')}>API reference</Link> under **Filters** and **Evidence** for complete request and response schemas.

You can update a manual filter with `PUT /api/filters/{id}`. Send its name, expression and `sspId` again: omitting or nulling `sspId` clears the SSP binding. `POST /api/filters/import` instead accepts multipart `files` containing arrays of dashboard objects with `name`, `filter` and optional `controls` entries (`catalog_id`, `id`). That importer creates global filters; use create/update for SSP binding. Check each file's result: an HTTP success may include failed files, and missing control references are skipped.

### Preview and scope provider work

Check whether support is enabled:

```sh
curl --fail-with-body -sS -H "Authorization: Bearer $TOKEN" "$API/dashboard-suggestions/config"
```

Set a shell prefix for this SSP:

```sh
export SSP_API="$API/oscal/system-security-plans/$SSP_ID"
```

Inspect available canonical label sets and copy the `hash` of a set you intend to use:

```sh
curl --fail-with-body -sS -H "Authorization: Bearer $TOKEN" "$SSP_API/dashboard-suggestions/label-sets"
```

In the next-release source, label pickers restrict evidence to the SSP scope when scope entries exist. Preview and generation combine that SSP scope with your requested `label-filter`. Without scope entries, evidence selection remains unrestricted unless you supply a filter. The `label-keys` and `label-values` pickers accept `ignoreScope=true` for scope editing; the label-set picker does not. Scope the run explicitly. The `label-keys` and `label-values?key=environment&query=fictional` endpoints under `dashboard-suggestions` help construct a label filter.

Save `run.json`, replacing the catalog UUID and hash with actual values. Control keys have the format `<catalog-uuid>:<control-id>`. Request fields use kebab-case; responses use camelCase.

```json
{
  "supersede-pending": false,
  "scope": {
    "control-keys": ["22222222-2222-4222-8222-222222222222:ac-3"],
    "label-set-hashes": ["<hash-from-label-sets>"],
    "label-filter": {
      "scope": {
        "condition": {"label": "environment", "operator": "=", "value": "fictional-demo"}
      }
    }
  }
}
```

Omitted or empty control/hash lists select all available values in that dimension. `label-filter` restricts evidence before label sets are gathered. Unknown keys or hashes produce `422`. Optional `constraints` support `mandatory-labels` and `excluded-labels` (selectors with `key` and optional `value`), `only-action` (`new_filter` or `extend_filter`) and `only-controls-without-filters` (a boolean).

```sh
curl --fail-with-body -sS -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' --data-binary @run.json "$SSP_API/dashboard-suggestions/preview"
```

Preview returns `plannedCalls`, `controlCount`, `labelSetCount`, `maxCallsPerRun` and `exceedsLimit` without creating a run or contacting the provider. Planned calls equal the number of control chunks multiplied by the number of label-set chunks. Narrow the scope if `exceedsLimit` is true.

### Generate and poll

Start with the same request:

```sh
curl --fail-with-body -sS -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' --data-binary @run.json "$SSP_API/dashboard-suggestions/generate" > run-response.json
```

A successful start returns `202` and a run ID:

```sh
export RUN_ID="$(jq -r '.data.id' run-response.json)"
```

Poll this endpoint until `status` is `completed` or `failed`, spacing requests to suit your integration:

```sh
curl --fail-with-body -sS -H "Authorization: Bearer $TOKEN" "$SSP_API/dashboard-suggestion-runs/$RUN_ID"
```

Inspect `completedCells`, `failedCells`, `plannedCalls`, `suggestionCount`, `error`, token totals and `stats`. `pending` and `running` are in-progress states. `dashboard-suggestion-runs/latest` retrieves the latest run. One active run per SSP is allowed; another start returns `409`. Setting `supersede-pending=true` supersedes pending suggestions in the selected scope; it does not bypass the active-run restriction.

### Inspect and decide suggestions

```sh
curl --fail-with-body -sS -H "Authorization: Bearer $TOKEN" "$SSP_API/dashboard-suggestions?status=pending" | jq --arg run "$RUN_ID" '.data[] | select(.runId == $run)'
```

Review each mapping's `controlId`, `controlCatalogId`, `labelSet`, `proposedFilterLabelSet`, `proposedFilterName`, `targetFilterId`, `confidence` and `reasoning`. Compare the proposed expression to actual evidence and control intent. The `dashboard-suggestions/control-results` endpoint distinguishes `matched` from `no_match`; a completed run can legitimately return no suggestions.

Save IDs you choose to accept as `accept.json`:

```json
{"ids": ["33333333-3333-4333-8333-333333333333"]}
```

```sh
curl --fail-with-body -sS -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' --data-binary @accept.json "$SSP_API/dashboard-suggestions/accept"
```

Acceptance creates an SSP-bound filter from the proposed labels, or reuses a same-SSP filter with the same canonical labels, and links the selected controls. It marks the suggestions accepted and records decision events. It does not change evidence or control implementation text. Read `data.acceptedFilterIds` and re-read compliance results to verify the dashboard.

For a different pending suggestion, save `reject.json` with its real ID and reason:

```json
{"ids": ["44444444-4444-4444-8444-444444444444"], "reason": "Evidence belongs to a different system"}
```

```sh
curl --fail-with-body -sS -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' --data-binary @reject.json "$SSP_API/dashboard-suggestions/reject"
```

Rejection records the decision without creating a filter. To revise a pending group before acceptance, save `edit.json`:

```json
{
  "ids": ["55555555-5555-4555-8555-555555555555"],
  "proposed-filter-name": "Fictional demo access evidence",
  "proposed-filter-label-set": {"environment": "fictional-demo", "type": "access-review"}
}
```

```sh
curl --fail-with-body -sS -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' --data-binary @edit.json "$SSP_API/dashboard-suggestions/edit-group"
```

Edits mark rows as user-edited; they do not accept them. The endpoint also supports `add-control-keys` and `remove-ids` for control membership. User-provided labels bypass the generated-label subset check, so verify the resulting filter matches evidence before accepting. Decisions require pending suggestions from the same SSP; a stale decision returns `409`. Inspect audit history with `GET /api/oscal/system-security-plans/{id}/dashboard-suggestions/{suggestionId}/events`.

### Generalize filters

```sh
curl --fail-with-body -sS -X POST -H "Authorization: Bearer $TOKEN" "$SSP_API/dashboard-suggestions/generalize"
```

This synchronous operation returns a run with `candidates` and `inserted` counts. Set `RUN_ID` to the returned run ID and use the client-side `runId` selection shown above, then review, accept or reject the pending suggestions. Generalization considers SSP filters that share control intent and differ by exactly one allowed instance key, such as `region`. It preserves meaning-bearing `_policy` and `type`. Review the broader evidence scope and `sourceFilterIds` before accepting an `isGeneralization` proposal.

Acceptance creates or reuses the generalized SSP filter and moves the accepted controls off its source filters onto it. The source filter records remain. Generalization does not call the AI provider, so it does not require provider generation to be enabled.

## Troubleshooting

| Symptom | What to check |
|---|---|
| Feature probe says `enabled: false` | Set `CCF_AI_ENABLED=true` and restart. The probe reports the feature flag, not provider credentials or worker readiness. Manual filters still work. |
| Generate returns `503`, or work stays pending | Enable `CCF_WORKER_ENABLED`, confirm AI worker registration and a working PostgreSQL/River queue, and inspect API startup logs. |
| Preview/generate returns `422` for empty input | Link profiles with resolved controls to the SSP; submit labeled evidence. Check your label filter and requested hashes. Check both the requested label filter and any configured SSP evidence scope. |
| Provider key/model/endpoint or timeout error | Check the configured key, model availability, endpoint and outbound network access. Inspect run and cell errors. A model default in the binary does not guarantee provider availability. |
| Frequent provider rate limits | Reduce `CCF_AI_QUEUE_WORKERS` or narrow scope. The worker snoozes rate-limited cells using provider retry guidance and jitter, up to 20 snoozes. Other job failures have up to three job attempts. |
| Planned-call limit exceeded (`422`) | Preview, reduce selected controls/label sets, or deliberately raise `CCF_AI_MAX_CALLS_PER_RUN`. It bounds planned cells, not cost or retry requests. |
| Fewer suggestions than expected | Check the insertion cap, validation/exclusion counts in `stats`, existing mappings and control results. Generated output is variable. Smaller chunks can help with truncated structured output; the stock CLI does not expose `ai_max_output_tokens`. |
| Decision returns `409` | Refresh pending suggestions: another decision or superseding run may have changed them. |
| Dashboard has no data or overly broad results | Compare the stored expression to actual evidence labels. Check the SSP binding and control association. The single-label example is deliberately broad; add `AND` conditions for your actual evidence intent. |

Admins can inspect `GET /api/admin/ai-diagnostics/summary`, `/api/admin/ai-diagnostics/runs` and `/api/admin/ai-diagnostics/runs/{runId}`. These expose feature/worker health, queue statistics, per-cell errors, mapping counts and token/cache usage. They help distinguish disabled support, queued work, provider failures and valid zero-output results.

## Related

- <Link to={useBaseUrl('/api/0.21/reference')}>API reference</Link>: Filters, Evidence, Dashboard Suggestions and AI Diagnostics.
- [Configuration reference](./configuration.md)
- [Authentication and permissions](./authentication-and-permissions.md)
- [Evidence artifacts and Rego playback](./evidence-and-playback.md)
