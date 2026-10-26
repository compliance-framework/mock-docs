---
title: Agent management
description: Enroll authenticated agents, preview remote configuration and verify instance convergence.
sidebar_position: 5
---

# Agent management

Operators use the Continuous Compliance Framework (CCF) API to enroll CCF Agents and manage their desired configuration. This guide covers service-account credentials, safe configuration changes and verification that running instances apply them.

## How it works

An agent record represents an identity with service-account keys. Each running process reports a separate instance UUID. An authenticated CCF Agent polls an overlay: an RFC 7396 JSON merge patch over its local configuration file. Configuration keys inside the overlay use `snake_case`; API envelope fields such as `applied-revision` use `kebab-case`.

The effective configuration is `MergePatch(file, overlay)`. An omitted key keeps the file value; `null` removes it so the default applies. The initial desired revision is `0`, with overlay `{}`. Each changed save appends an immutable revision. A save changes desired state, not running state: each instance decides whether to apply it and reports the result.

### Host controls

The host sets `remote_config` locally. An overlay cannot change `api`, `daemon` or `remote_config`, even to `null`.

| Mode | Behavior |
|---|---|
| `off` | Does not fetch or report remote configuration. Default without API credentials; without credentials the agent always uses this mode. |
| `report` | Reports configuration but never applies an overlay. Default with credentials. |
| `apply_safe` | Applies only revisions whose changes are all safe. |
| `apply_all` | Also applies unsafe changes, but still rejects forbidden changes or invalid merged configuration. |

Logging, evidence settings, plugin schedules, labels, policy inputs and disabling a plugin are safe changes. Removing plugins or policies reduces scope and is safe. Thus `apply_safe` can change policy pass/fail inputs or stop collection.

A new plugin or policy source is safe when an enabled plugin in the local file already uses it, or it matches `remote_config.trusted_sources`. These case-sensitive globs use `path.Match`: `*` does not cross `/`. Local paths are never trusted. Introducing a local source is forbidden unless the host uses both `apply_all` and `allow_local_sources: true`.

Plugin config changes are safe when the key matches `remote_config.overridable_config_flags` (for example `fictional-checker:region`); otherwise they are unsafe. New `${env:NAME}` references are unsafe. References to `CCF_API_AUTH_*` are forbidden. Re-enabling a locally disabled plugin is unsafe unless its source is trusted; its sources and environment references still undergo classification. See the [released CCF Agent configuration reference](/agent/0.9/configuration) for all host settings and defaults.

## Before you start

This procedure targets API **v0.21.1**. Use a CCF Agent with remote-configuration support, documented in the released `0.9` line. Older agents may send heartbeats without configuration reports.

You need a running API, its database migrations, `curl` and `jq` for these examples, and a user bearer token from [Authentication and permissions](./authentication-and-permissions.md). Keep user and agent tokens separate:

| Caller and operation | Permission |
|---|---|
| User: create, update or delete agents; create, read or revoke keys | `admin:manage` |
| User: list/read agents, desired configuration, revisions and instances | `agent:read` |
| User: preview, save or revert configuration | `agent:configure` |
| Agent: poll its own configuration or report its own instances | `agent:sync` |
| Agent: send authenticated heartbeats | `heartbeat:ingest` |

The default `builtin` authorization driver requires the admin check for user access to `agent:*`. With Cedar, grant the actions your operator role needs. Configuration sync always requires an agent JWT, even when public agent ingestion endpoints are enabled. A user JWT cannot substitute for one.

## Enable it

### Enroll an agent and issue a key

The examples use a fictional API address. Set `ADMIN_TOKEN` to your user token. Substitute returned IDs and credentials; the API creates them, not the client.

```sh
export API_URL='https://ccf-api.example.test'
```

Create the agent identity (the service account). Success returns `201` with `data.id`:

```sh
AGENT_ID=$(curl --fail-with-body -sS "$API_URL/api/admin/agents" \
  -H "Authorization: Bearer $ADMIN_TOKEN" -H 'Content-Type: application/json' \
  --data '{"name":"fictional-lab-agent","description":"Fictional documentation lab","is-active":true}' \
  | jq -er '.data.id')
```

Create a key. Supply a future RFC 3339 `expires-at`, or explicitly use `never-expires: true`; do not combine them. This example expires in 2099; choose your organization's rotation interval for an actual deployment.

```sh
KEY_JSON=$(curl --fail-with-body -sS "$API_URL/api/admin/agents/$AGENT_ID/keys" \
  -H "Authorization: Bearer $ADMIN_TOKEN" -H 'Content-Type: application/json' \
  --data '{"name":"fictional-lab-key","expires-at":"2099-01-01T00:00:00Z"}')
```

The `201` response includes `data.client-id` and `data.client-secret`. The secret appears only at creation; store it securely. Later key reads omit it.

```sh
CLIENT_ID=$(printf '%s' "$KEY_JSON" | jq -er '.data["client-id"]')
```

```sh
CLIENT_SECRET=$(printf '%s' "$KEY_JSON" | jq -er '.data["client-secret"]')
```

Exchange the key for a bearer token using HTTP Basic authentication. The response is not wrapped in `data`; `expires_in` is `86400` seconds.

```sh
AGENT_TOKEN=$(curl --fail-with-body -sS -X POST "$API_URL/api/auth/agent/token" \
  --user "$CLIENT_ID:$CLIENT_SECRET" | jq -er '.access_token')
```

The token endpoint also accepts form fields `client_id` and `client_secret`. The CCF Agent obtains and refreshes its token from its configured credentials.

### Configure the host

Set `api.auth.client_id` and `api.auth.client_secret` through the host's credential mechanism (or `CCF_API_AUTH_CLIENT_ID` and `CCF_API_AUTH_CLIENT_SECRET`). Start in `report` mode to inspect the reported base before opting into application. For the safe logging change below, set this block locally when ready:

```yaml
remote_config:
  mode: apply_safe
  poll_interval: 60s
```

Keep the existing `api`, `daemon` and plugin configuration. This is a fragment, not a complete agent file. The polling minimum is `15s`. Follow the [CCF Agent configuration reference](/agent/0.9/configuration) for credentials, trusted sources, safe config flags and persistent instance state.

## Use it

### Report a base configuration

A real CCF Agent sends reports itself. The following synthetic report demonstrates the wire format for a fictional instance before the first save. It does not start a process or prove that a process applied configuration. The all-zero digest is a format-valid placeholder; real reports use the agent-computed SHA-256 digest of its canonical redacted effective configuration with the API block excluded.

```sh
INSTANCE_ID='11111111-1111-4111-8111-111111111111'
```

```sh
curl --fail-with-body -i -X PUT "$API_URL/api/agent/instances/$INSTANCE_ID/config-report" \
  -H "Authorization: Bearer $AGENT_TOKEN" -H 'Content-Type: application/json' \
  --data '{"hostname":"fictional-lab-node","agent-version":"0.9.0","mode":"apply_safe","daemon":true,"applied-revision":0,"attempted-revision":0,"status":"applied","error":null,"base":{"verbosity":0,"plugins":{}},"effective":{"verbosity":0,"plugins":{}},"effective-digest":"sha256:0000000000000000000000000000000000000000000000000000000000000000","remote-config":{"mode":"apply_safe","poll_interval":"60s","trusted_sources":[],"overridable_config_flags":[],"allow_local_sources":false}}'
```

Success is `204`. Use this synthetic instance only in a disposable lab. For a real deployment, use the instance UUID and reports produced by the running agent. An authenticated heartbeat with `config_digest` can also register an instance and record `config_revision`; heartbeats alone do not provide the base needed for merged validation.

### Read, preview and save

Read the current desired configuration with your user token:

```sh
curl --fail-with-body -i "$API_URL/api/admin/agents/$AGENT_ID/config" \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

For a new identity, expect `data.revision: 0`, `data.overlay: {}` and `ETag: "0"`. Preview a safe logging change:

```sh
curl --fail-with-body -sS "$API_URL/api/admin/agents/$AGENT_ID/config/preview" \
  -H "Authorization: Bearer $ADMIN_TOKEN" -H 'Content-Type: application/json' \
  --data '{"overlay":{"verbosity":1}}'
```

A `200` preview can contain validation failures. Inspect `data.overlay-errors` and each instance's `errors`, `warnings`, `validated`, `changes`, `diff-vs-current`, `will-apply` and `will-apply-reason`. The example instance should classify this as safe and set `will-apply: true`. Report-mode instances return `mode-report`; off-mode instances return `mode-off`. Preview never saves or applies anything.

Save the same overlay with the ETag from the admin read. For this new identity:

```sh
curl --fail-with-body -i -X PUT "$API_URL/api/admin/agents/$AGENT_ID/config" \
  -H "Authorization: Bearer $ADMIN_TOKEN" -H 'Content-Type: application/json' \
  -H 'If-Match: "0"' \
  --data '{"overlay":{"verbosity":1},"comment":"Enable debug logging in the fictional lab"}'
```

Expect `201`, `data.revision: 1` and `ETag: "1"`. Saving a semantically unchanged overlay returns `200` without a new revision. Each save replaces the entire stored overlay; it does not merge with the previous overlay. Include every desired override in the candidate.

The admin `If-Match` accepts a plain revision number, a quoted number or a weak quoted number. Missing or invalid values return `428`. The agent-facing `GET /api/agent/config` instead returns an **opaque** ETag: send it back verbatim in `If-None-Match`. A matching poll returns `304`; never construct this agent ETag from a revision number.

### Verify convergence

Let the real agent poll and report, then read instances with your user token:

```sh
curl --fail-with-body -sS "$API_URL/api/admin/agents/$AGENT_ID/instances?page=1&limit=25" \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

`meta.desired-revision: 1` confirms the saved target. For each apply-mode instance, check `applied-revision: 1`, `status: applied`, `sync-status: in-sync`, `stale: false` and `report-stale: false`. `attempted-revision: 1` with `applied-revision: 0` and `status: rejected` shows an attempted change, not convergence. An instance that has not attempted the target can show `pending`. Report/off modes have `sync-status: not-applicable`; a heartbeat-only instance can remain `unknown`.

Inspect a particular instance's redacted base, effective config, reason and error:

```sh
curl --fail-with-body -sS "$API_URL/api/admin/agents/$AGENT_ID/instances/$INSTANCE_ID" \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

`report-stale` means the heartbeat digest differs from the last configuration report. `stale` means the instance has not been seen within the freshness window. Check both; a recent heartbeat does not make an older configuration report current. Counts in `meta.counts` cover all instances, not only this page. Follow `meta.total-pages`; the list defaults to 25 and caps `limit` at 25.

### Handle concurrent edits and validation errors

Repeating the first save with `If-Match: "0"` after revision `1` exists returns `409`. Its `errors.current-revision` is `1`. Read the current overlay and ETag again, reconcile your change, preview and retry with that ETag. Do not blindly overwrite another operator's revision.

For a validation failure, try a locked-key change against the current revision in a lab:

```sh
curl -i -X PUT "$API_URL/api/admin/agents/$AGENT_ID/config" \
  -H "Authorization: Bearer $ADMIN_TOKEN" -H 'Content-Type: application/json' \
  -H 'If-Match: "1"' --data '{"overlay":{"daemon":false}}'
```

Expect `422` with `errors.body: "configuration overlay is invalid"`, an `errors.overlay` list and an `errors.instances` list. No revision is created. Unknown keys, invalid types, schedules, plugin names, forbidden environment references, NUL characters or an overlay larger than 256 KiB compact also fail validation. Malformed envelopes return `400`; non-JSON content types return `415`; admin bodies above 1 MiB return `413`. Comments allow at most 2000 characters and no NUL.

Saves validate merged configuration against every fresh apply-mode instance with a reported base. With none, validation falls back to the most recently reported apply-mode instance, however old. With none at all, only overlay validation runs (`standalone: true` in preview). Report-mode instances do not enter this validation set. Errors introduced by the overlay block a save; errors already in the local base appear as non-blocking warnings.

Preview includes fresh and stale instances, with validated instances first. It is limited to 50 instances and 16 MiB of reported configuration; check `omitted-instances`. A save still validates the full validation set. Successful API validation does not override the host's safety decision.

### Inspect history and revert

List immutable revisions, newest first; the list omits overlays. Pagination defaults to `page=1`, `limit=50` and allows at most 100 per page:

```sh
curl --fail-with-body -sS "$API_URL/api/admin/agents/$AGENT_ID/config/revisions?page=1&limit=50" \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

Read an earlier revision's overlay and metadata:

```sh
curl --fail-with-body -sS "$API_URL/api/admin/agents/$AGENT_ID/config/revisions/1" \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

To restore revision `1` after another edit, read the current admin ETag again and put its exact value in `CURRENT_ETAG`. Then:

```sh
curl --fail-with-body -i -X POST "$API_URL/api/admin/agents/$AGENT_ID/config/revisions/1/revert" \
  -H "Authorization: Bearer $ADMIN_TOKEN" -H 'Content-Type: application/json' \
  -H "If-Match: $CURRENT_ETAG" --data '{"comment":"Restore fictional lab logging"}'
```

A changed revert appends the next revision with `revert-of: 1`; it does not rewind history. It uses the same concurrency and validation rules as save. An unchanged revert returns `200` without a revision. To return to local-file configuration, save `{"overlay":{}}` with the current ETag. Verify convergence again after either operation.

### Manage instance lifetime

API defaults come from v0.21.1. The [API configuration reference](./configuration.md) covers deployment configuration.

| Setting | Purpose | Type | Default | Example |
|---|---|---|---|---|
| `CCF_AGENT_INSTANCE_ONESHOT_RETENTION` | Prune one-shot (`daemon: false`) instances after this unseen duration. *(new in 0.21)* | `duration` | `24h` | `48h` |
| `CCF_AGENT_INSTANCE_PRUNE_ENABLED` | Schedule pruning; requires the worker service. *(new in 0.21)* | `bool` | `true` | `true` |
| `CCF_AGENT_INSTANCE_PRUNE_SCHEDULE` | River cron with seconds first; pruning runs at most hourly. *(new in 0.21)* | `string` | `0 17 * * * *` | `0 30 * * * *` |
| `CCF_AGENT_INSTANCE_RETENTION` | Prune daemon or unknown instances after this unseen duration. *(new in 0.21)* | `duration` | `720h` | `168h` |
| `CCF_AGENT_INSTANCE_STALE_AFTER` | Reports or heartbeats within this duration keep instances fresh for validation and cap handling. *(new in 0.21)* | `duration` | `10m` | `15m` |
| `CCF_AGENT_MAX_INSTANCES` | Cap instances not yet eligible for pruning, per agent identity. *(new in 0.21)* | `int` | `500` | `250` |

Unset or non-positive duration/cap values use the defaults. At the cap, a new instance replaces the oldest stale instance. If every counted instance is fresh, a new report gets `409` and its heartbeats are not recorded. Existing instances can still report. Back off instead of generating more instance IDs. Pin one-shot instance identity as described in the [CCF Agent reference](/agent/0.9/configuration). Deleting an agent deletes its instances and revision history.

### Keep secrets on the host

The agent receives its overlay verbatim. Admin current/revision reads return literal overlays only to callers who also hold `agent:configure`; other readers get secret-like keys and values masked as `••••`. If that permission check cannot be evaluated, masking is the default. Revision lists never contain overlays.

Instance `base` and `effective` documents and preview effective configs are always redacted. The agent redacts them first; the API re-redacts them as a best effort. It also masks secret-like content in report errors, warnings, plugin sources, unsafe change values and `remote-config` strings. This is not a guarantee that arbitrary free text cannot contain secrets. Stored revisions retain literal values, and editors can read them. Prefer `${env:NAME}` placeholders in `plugins.<name>.config` values; supply the value on the host. Never reference `CCF_API_AUTH_*`.

## Troubleshooting

| Symptom | What to check |
|---|---|
| Token acquisition returns `401` | Supply both key credentials; confirm the client ID and secret. Missing credentials and invalid credentials return `401`. |
| Token acquisition returns `403` | Check for an expired or revoked key or an inactive agent. Create a replacement key before revoking the old one with `DELETE /api/admin/agents/{id}/keys/{keyId}`. |
| Sync returns `401` or `403` | Use an agent JWT for that identity and grant `agent:sync`. Public ingestion does not enable anonymous config sync. Admin operations need a user JWT and the permissions above. |
| No configuration report, or `unsupported-by-agent` | Check the agent version and plugin `lib-version` reports. Older agents may not implement remote configuration; newer overlays can use unsupported fields. Upgrade the agent or remove incompatible fields, then preview again. A `404` on the agent config route can indicate an API without remote configuration support. |
| Saved revision remains pending | Check host credentials, mode, polling, API connectivity and freshness. A successful save does not make a report-mode agent apply it. |
| `rejected`, `unsafe-changes` or `forbidden-changes` | Inspect instance `unsafe`, preview `changes` and `will-apply-reason`. Change the overlay or adjust trusted sources/config flags locally after reviewing the scope. Locked blocks remain forbidden in every apply mode. |
| `failed`, `invalid-config`, `env-missing` or `download-failed` | Inspect `reason`, `error` and `warnings`; fix host environment values, merged configuration or source access. Re-preview and check a new real report. |
| Report returns `409` | The instance cap is full of fresh instances. Reuse a stable instance identity, back off, or review the API cap. Verify the pruning worker runs; pruning has an hourly maximum frequency. This differs from a save's stale-revision `409`. |
| Report returns `400`, `413` or `415` | Use a UUID, valid mode/status and `sha256:` plus 64 lowercase hex digits; send object-valued base/effective configs with no NUL. Use JSON and stay within the 4 MiB report limit. |
| Details are incomplete | Check `truncated`. The API bounds report lists, strings and summaries (3 MiB encoded per instance); long `remote-config` list entries are dropped. Request instance detail for base/effective configs and inspect host logs. |

## Related

- [API overview](./index.md)
- [Authentication and permissions](./authentication-and-permissions.md)
- [API configuration reference](./configuration.md)
- [Released CCF Agent configuration reference](/agent/0.9/configuration)
