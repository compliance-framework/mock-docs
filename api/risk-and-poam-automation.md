---
title: Risk and POA&M automation
description: Configure risk review jobs, evidence reconciliation and POA&M remediation automation.
---

# Risk and POA&M automation

As an operator, you can configure the Continuous Compliance Framework (CCF) API to track risks from evidence through remediation. This guide covers the next-release API source, checked against `main` at `dc4b3f9`, and a developer example that promotes a risk to a plan of action and milestones (POA&M) item.

## How it works

Risk ingestion and periodic scans serve different purposes:

- When evidence changes, a worker matches `not-satisfied` evidence against risk templates and SSP evidence filters. It creates or updates risks, links evidence, and updates `last-seen-at`. Matching failures can reopen a `remediated` risk to `open`; closed risks are excluded from reuse. This behavior does not depend on `auto_reopen_enabled`.
- When evidence becomes satisfied or no longer matches the template's violations, the worker removes the corresponding evidence link. When no links remain, it moves a nonterminal risk to `remediated`. For `risk-accepted`, it records `evidence_recovered` without changing status. A remaining evidence link prevents automatic remediation.
- The optional `evidence_reconciliation_enabled` scan finds duplicate nonclosed risks by nonempty deduplication key. It retains the oldest and closes the others. It does not replay all evidence or replace evidence ingestion.
- Review reminders notify owners of accepted risks with a review deadline in the next 30 days. Overdue escalation scans accepted risks whose review deadline has passed. With `auto_reopen_enabled`, that scan also queues reopening after `auto_reopen_threshold_days` overdue: the review changes the risk to `investigating` and clears its deadline and acceptance justification.
- The stale-risk scan sends reminders for `open`, `investigating`, `mitigating-planned` and `mitigating-implemented` risks last seen at least 30 days ago. It does not change their status.
- Risk digests summarize new risks, overdue action, stale risks, and accepted-risk reviews that are due or overdue. POA&M digests group new, overdue, approaching-deadline and stale items, plus milestones due soon. `daily` covers the previous UTC calendar day; `weekly` covers the previous complete Monday-based UTC week. The window does not change the cron schedule. These jobs do not change status.

POA&M deadline reminders select `open` and `in-progress` items with a future planned completion date within `reminder_window_days`. The overdue scanner moves those statuses to `overdue` once the date has passed, then queues notifications. Moving the date forward does not automatically reverse `overdue`; update the status explicitly. Milestone checks notify the parent item's primary owner when an `open` or `in-progress` milestone is past its planned completion date and the parent is not `completed`. They do not complete milestones or change the parent status.

State changes and notification delivery are separate. An overdue transition can succeed with no recipient or with delivery unavailable. Risk reminders use the primary owner and user owner assignments; POA&M reminders and digests use the primary owner. A milestone's free-text responsible party is not a notification recipient. Configure channels and subscriptions through [Notifications](./notifications.md).

## Before you start

- Run the API with PostgreSQL and its migrations applied. Keep `CCF_WORKER_ENABLED=true` (default) so River processes evidence, scans and delivery jobs. These jobs do not require the workflow scheduler to be enabled.
- Have an SSP and user UUIDs for your test records. Evidence-driven creation also needs risk templates with matching violation IDs and evidence labels that match SSP filters.
- Use an authenticated user token. Creation requires `create` on `risk`; reading requires `read`; updates, acceptance and reviews require `update`; promotion requires `promote`. Completing remediation requires `update` on `poam_item`. The bundled Cedar `admin`, `contributor` and `auditor` roles grant these operations; `viewer` cannot mutate them. The default `builtin` driver permits authenticated users on these resources. Check your configured policy through [Authentication and permissions](./authentication-and-permissions.md).
- Configure delivery providers and user destinations using [Notifications](./notifications.md). Set `CCF_WEB_BASE_URL` to your UI address for notification links.

For interactive register operations, use the released [UI risk guide](../ui_versioned_docs/version-2.12/features/risks.md). This page uses API requests and does not require a UI recording.

## Enable it

### File loading and environment overrides

The executable reads `risk.yaml` and `poam.yaml` from its working directory. Relative paths resolve there, including inside a container. You can select a different risk file with `CCF_RISK_CONFIG=/etc/ccf/risk.yaml`; an absolute path avoids working-directory ambiguity.

In this source revision, `cmd/root.go` binds `risk_config` but does not bind `poam_config`, and does not enable global automatic environment lookup. Although `internal/config/poam.go` mentions `CCF_POAM_CONFIG`, the executable does **not** support that path override. Mount `poam.yaml` in the working directory, or use the individual `CCF_POAM_*` overrides below.

Both file loaders support per-setting environment overrides: uppercase the YAML key and prefix it with `CCF_RISK_` or `CCF_POAM_`. For example, `open_digest_enabled` becomes `CCF_RISK_OPEN_DIGEST_ENABLED` or `CCF_POAM_OPEN_DIGEST_ENABLED`. Environment values take precedence over file values, then defaults. Keys in each file are top-level, without a `risk:` or `poam:` wrapper. Restart the API after changes.

All job enable flags and risk auto-reopening default to `false`. Cron schedules have six fields: second, minute, hour, day of month, month, day of week. Risk validation also accepts descriptors such as `@daily`; POA&M validation requires six fields. Use UTC for the API process for the clock times below; digest boundaries and deadline comparisons use UTC. Periodic jobs do not run immediately at startup, and queue uniqueness can suppress repeated runs within daily or weekly periods.

A missing file uses defaults plus environment values. An unreadable or malformed file, invalid type or failed validation produces a warning and resets that **entire** risk or POA&M configuration to disabled defaults. The API continues running. Enabled schedules are validated; a bad schedule on a disabled job is ignored. Risk digest windows accept `daily` or `weekly` after trimming and case normalization; a nonempty invalid window fails even with the digest disabled. The reopen threshold must be nonnegative, and positive when reopening is enabled. POA&M always requires an exact `daily` or `weekly` window and positive `reminder_window_days`, even when jobs are disabled.

### Risk settings (`risk.yaml`)

| Setting | Purpose | Type | Default | Example |
|---|---|---|---|---|
| `auto_reopen_enabled` | Reopen overdue accepted risks through the enabled overdue escalation scan. | `bool` | `false` | `true` |
| `auto_reopen_threshold_days` | Days after the accepted-risk review deadline before reopening. | `int` | `30` | `14` |
| `evidence_reconciliation_enabled` | Reconcile duplicate nonclosed risks. | `bool` | `false` | `true` |
| `evidence_reconciliation_schedule` | Duplicate reconciliation schedule; daily at 10:30. | `string` | `0 30 10 * * *` | `0 30 10 * * *` |
| `open_digest_enabled` | Queue owner risk digests. | `bool` | `false` | `true` |
| `open_digest_schedule` | Risk digest schedule; daily at 11:00. | `string` | `0 0 11 * * *` | `0 0 11 * * 1` |
| `open_digest_window` | Digest period: `daily` or `weekly`. | `string` | `daily` | `weekly` |
| `review_deadline_reminder_enabled` | Remind accepted-risk owners within the fixed 30-day window. | `bool` | `false` | `true` |
| `review_deadline_reminder_schedule` | Review reminder schedule; daily at 08:00. | `string` | `0 0 8 * * *` | `0 0 8 * * *` |
| `review_overdue_escalation_enabled` | Notify owners of overdue accepted-risk reviews and optionally reopen risks. | `bool` | `false` | `true` |
| `review_overdue_escalation_schedule` | Escalation schedule; daily at 09:00. | `string` | `0 0 9 * * *` | `0 0 9 * * *` |
| `stale_risk_scanner_enabled` | Remind owners of active risks last seen at least 30 days ago. | `bool` | `false` | `true` |
| `stale_risk_scanner_schedule` | Stale-risk schedule; Monday at 10:00. | `string` | `0 0 10 * * 1` | `0 0 10 * * 1` |

### POA&M settings (`poam.yaml`)

| Setting | Purpose | Type | Default | Example |
|---|---|---|---|---|
| `deadline_reminder_enabled` | Remind owners of approaching remediation deadlines. | `bool` | `false` | `true` |
| `deadline_reminder_schedule` | Deadline reminder schedule; daily at 08:00. | `string` | `0 0 8 * * *` | `0 0 8 * * *` |
| `milestone_overdue_enabled` | Notify owners about incomplete, overdue milestones. | `bool` | `false` | `true` |
| `milestone_overdue_schedule` | Milestone check schedule; Monday at 10:00. | `string` | `0 0 10 * * 1` | `0 0 10 * * 1` |
| `open_digest_enabled` | Queue owner POA&M digests. | `bool` | `false` | `true` |
| `open_digest_schedule` | POA&M digest schedule; daily at 07:00. | `string` | `0 0 7 * * *` | `0 0 7 * * 1` |
| `open_digest_window` | Digest period: `daily` or `weekly`. | `string` | `daily` | `weekly` |
| `overdue_transition_enabled` | Change overdue `open` or `in-progress` items to `overdue`. | `bool` | `false` | `true` |
| `overdue_transition_schedule` | Overdue transition schedule; daily at 09:00. | `string` | `0 0 9 * * *` | `0 0 9 * * *` |
| `reminder_window_days` | Look-ahead window in days for future completion deadlines. | `int` | `30` | `14` |

### Start with reminders

Save this as `risk.yaml` in the API working directory. Omitted settings retain their defaults; auto-reopening stays disabled.

```yaml
review_deadline_reminder_enabled: true
review_overdue_escalation_enabled: true
stale_risk_scanner_enabled: true
evidence_reconciliation_enabled: true
open_digest_enabled: true
open_digest_window: daily
```

Save this as `poam.yaml` in the same directory. Enabling overdue transitions changes records, so choose it deliberately.

```yaml
deadline_reminder_enabled: true
reminder_window_days: 14
overdue_transition_enabled: true
milestone_overdue_enabled: true
open_digest_enabled: true
open_digest_window: daily
```

Keep workers enabled and start the API from that directory:

```sh
CCF_WORKER_ENABLED=true api run
```

## Use it

### Create, review and promote a risk

The example uses fictional data. Set `API_URL` to your API origin, `TOKEN` to an authenticated user token, and `SSP_ID` and `OWNER_ID` to existing test records. The URL includes `/api` in each request.

```sh
export API_URL=https://api.example.com
```

Create an investigating risk (HTTP `201`). Copy `data.id` from the response into `RISK_ID`.

```sh
curl --fail-with-body "$API_URL/api/risks" \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  --data "{\"title\":\"Fictional backup recovery gap\",\"description\":\"Validate recovery for a test service\",\"ssp-id\":\"$SSP_ID\",\"primary-owner-user-id\":\"$OWNER_ID\",\"status\":\"investigating\"}"
```

Record a reassessment (HTTP `200`). It appends a review and updates likelihood, impact and last-reviewed time; the risk stays `investigating`.

```sh
curl --fail-with-body "$API_URL/api/risks/$RISK_ID/review" \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  --data '{"decision":"reassess","likelihood":"moderate","impact":"high","notes":"Recovery exercise confirms remediation is needed."}'
```

Promote it (HTTP `201`), and copy the returned `data.id` into `POAM_ID`. Choose a future deadline for your test.

```sh
curl --fail-with-body "$API_URL/api/risks/$RISK_ID/promote-to-poam" \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  --data '{"deadline":"2030-12-01T00:00:00Z","resourceRequired":"One fictional recovery exercise","milestones":[{"title":"Verify restoration in the test environment"}]}'
```

Promotion creates an `open` item with source `risk-promotion`, links it to the risk, inherits its owner unless overridden, and changes the risk to `mitigating-planned`. Existing remediation-template tasks become milestones before extra request milestones. The transaction records `status_changed` and `poam_promoted` events. Promotion requires `investigating` and no linked noncompleted POA&M item; otherwise it returns HTTP `422`. The request's `resourceRequired` and optional `primaryOwnerUserId` use camel case, unlike the risk creation fields.

Acceptance is an alternative to promotion: `POST /api/risks/{id}/accept` requires an investigating risk, a nonempty `justification`, and a future `review-deadline`. It sets `risk-accepted`. Reviews can `extend` an accepted risk with `next-review-deadline`, or `reopen` an accepted or mitigating risk to `investigating`. A review decision of `implement` advances `mitigating-planned` to `mitigating-implemented`.

### Complete and verify remediation

After the work is verified, update the item (HTTP `200`):

```sh
curl --fail-with-body -X PUT "$API_URL/api/poam-items/$POAM_ID" \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  --data '{"status":"completed"}'
```

On the transition to `completed`, the handler advances linked risks from `mitigating-planned` to `mitigating-implemented` only if **all** their linked POA&M items are completed. It records `status_changed` and `poam_completed`. Other risk statuses remain unchanged. If advancing a linked risk fails, the handler logs `failed to advance linked risk statuses on POAM completion` but still returns success for the saved item; verify both records. This does not mark evidence satisfied, close the risk, or guarantee `remediated`; evidence resolution is a separate path.

### Verification checklist

- Confirm startup has no `Failed to load risk config` or `Failed to load poam config` warnings, and the worker service starts.
- Read `GET /api/risks/{id}`, `GET /api/risks/{id}/reviews` and `GET /api/risks/{id}/events` with a user allowed to read `risk`. Check the reassessment, promotion link and status events.
- Read `GET /api/poam-items/{id}` with read access to `poam_item`. Check its owner, deadline, risk links and milestones before and after completion.
- Wait for a configured scan with a test record inside its time window. Verify notification jobs and provider delivery separately from status changes. Do not expect an immediate scan on restart.
- For an evidence-created test risk, submit matching failing evidence, then satisfied evidence for the same stream. Check `GET /api/risks/{id}/evidence` for remaining links and the resulting events. A manual risk in the example has no automatic evidence association.

## Troubleshooting

| Symptom | What to check |
|---|---|
| All risk or POA&M jobs stay disabled | Read startup warnings. One invalid value resets that whole configuration to disabled defaults. Check six-field cron syntax, positive windows and threshold rules, then restart. |
| A mounted file has no effect | Verify the API working directory and mount permissions. Use `CCF_RISK_CONFIG` for the risk path; `CCF_POAM_CONFIG` is not bound in this executable. Check per-key environment overrides. |
| No scan or evidence processing | Keep `CCF_WORKER_ENABLED=true`, check worker startup and River jobs, then the individual job enable flag and next cron time. Daily/weekly uniqueness can suppress rapid repeat scans. |
| No recipients or notifications | Set risk primary/user assignment owners or the POA&M primary owner to existing users. Check their subscriptions, enabled channel/provider, delivery logs and `CCF_WEB_BASE_URL` using [Notifications](./notifications.md). An SSP owner or milestone responsible-party string alone is insufficient. |
| Accepted risk never reopens | Enable both overdue escalation and auto-reopening. Its accepted review deadline must be overdue by at least the configured threshold; evidence recurrence uses a different reopening path. |
| Evidence stays unresolved | Inspect all linked streams, their latest status, template violation matches and SSP filters. Satisfying one stream does not remove other links. Accepted risks record recovery without automatic status change. The duplicate reconciliation scan does not replay missing evidence jobs. |
| Promotion returns `422` | Move the risk through a permitted review/update to `investigating` and complete any active linked remediation item before retrying. |
| Completed remediation does not close a risk | Expect `mitigating-implemented` only from `mitigating-planned` and only after all linked items complete. Check unresolved evidence separately. |

## Related

- [UI risk guide](../ui_versioned_docs/version-2.12/features/risks.md): interactive register operations.
- [Notifications](./notifications.md): providers, subscriptions and delivery checks.
- [Evidence artifacts and Rego playback](./evidence-and-playback.md): evidence submission and diagnosis.
- [Authentication and permissions](./authentication-and-permissions.md): policy drivers and roles.
- [Configuration reference](./configuration.md): executable settings.
