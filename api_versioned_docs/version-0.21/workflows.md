---
title: Workflow scheduling and tasks
description: Configure workflow scheduling, create an SSP-bound workflow, complete dependent tasks and verify notifications.
sidebar_position: 8
---

# Workflow scheduling and tasks

Use the Continuous Compliance Framework (CCF) API to schedule recurring compliance work and collect evidence as users complete tasks. This guide is for operators and developers integrating with API `v0.21.1`; for screen-based steps, see the released UI `2.12` workflow guide.

## How it works

A **definition** describes steps, dependencies, responsible roles and required evidence. An **instance** binds a definition to a system security plan (SSP), a cadence and role assignments. Each **execution** creates step executions: the tasks users complete.

The River scheduler periodically finds active instances whose next scheduled time has arrived. It creates an execution and queues initialization on the `workflow` queue. Initialization makes independent tasks `pending` and dependent tasks `blocked`. Users move actionable tasks through `in_progress` to `completed`; completing prerequisites unblocks their dependents. CCF does not perform the compliance activity for the user.

### Cadence and deadlines

`scheduler_schedule` controls how often CCF checks for due instances. An instance's `cadence` controls when that instance is due. Supported cadences are `daily`, `weekly`, `monthly`, `quarterly`, `annually`, or `cron:` followed by six fields, for example `cron:0 0 9 * * *`. Named cadences advance by calendar days, weeks, months, quarters or years from the previous scheduled time. Cron cadences require seconds and do not accept descriptors such as `@daily` after `cron:`.

Creation with a nonempty cadence sets the first `next-scheduled-at` to the next occurrence after creation. An empty cadence creates no scheduled time; `suggested-cadence` on a definition does not populate the instance cadence. Scheduled executions carry a `period_label` and are deduplicated per instance and period. Period labels use a date for daily runs, an ISO week for weekly runs, a month, quarter or year for the other named cadences, and a timestamp for cron. A delayed scheduler uses the stored scheduled time for the execution deadline and advances the schedule from that time.

Execution grace days resolve from instance → definition → `grace_period_days`. A scheduled execution's `due_date` is its scheduled time plus those days. Step deadlines resolve from step → instance → definition → **seven days** in this release: the last fallback uses the code default even if you override the global configuration. Independent task deadlines start at execution start; dependent tasks receive their deadline when unblocked. `estimated-duration` is a planning estimate in minutes, not a scheduling timeout.

With overdue checks enabled, the scheduler marks late tasks and executions `overdue`. An overdue execution becomes `failed` after another effective execution grace period measured from `overdue-at`. Grace values must be nonnegative. When both are set, instance grace must be at least definition grace, and the sum of explicit step grace values cannot exceed definition grace.

## Before you start

- Run API `v0.21.1` with PostgreSQL and migrations applied. Keep `CCF_WORKER_ENABLED=true` (default). Workers process initialization, scheduling, overdue checks and delivery.
- Have an existing SSP UUID and a registered user UUID for the assignee. Authenticate with a bearer token; the transition handler resolves the actor from that token.
- For the lifecycle below, use `create` and `read` on `workflow-definition`, `workflow-step-definition`, `workflow-instance`, `role-assignment` and `workflow-execution`, plus `read` and `update` on `step-execution`. The bundled `contributor` role grants these. Task completion also requires an active assignment to the step's responsible role; an API role alone does not satisfy it.
- Notification diagnostics and system destinations require `admin:manage` (the bundled `admin` role). Personal subscription updates require update access to your own user resource. See [Authentication and permissions](./authentication-and-permissions.md).
- Configure delivery providers, destinations, subscriptions and `CCF_WEB_BASE_URL` using [Notifications](./notifications.md).

## Enable it

The API loads `workflow.yaml` from its working directory unless `CCF_WORKFLOW_CONFIG` selects another file. Missing files use defaults and environment overrides. Environment values override YAML values. Restart the API after changing configuration.

### File settings

These are all supported keys. Defaults below come from code, not the repository's sample file.

| Setting | Purpose | Type | Default | Example |
|---|---|---|---|---|
| `due_soon_enabled` | Register the due-soon reminder scan independently of workflow scheduling. | `bool` | `false` | `true` |
| `due_soon_schedule` | Cron schedule for due-soon scans. | `string` | `0 0 8 * * *` | `0 0 9 * * *` |
| `grace_period_days` | Fallback execution grace, in calendar days; minimum `0`. See step fallback above. | `int` | `7` | `14` |
| `overdue_check_enabled` | Check overdue tasks, overdue executions and expired execution grace during scheduler jobs. Requires `scheduler_enabled`. | `bool` | `true` | `true` |
| `scheduler_enabled` | Register the periodic workflow scheduler. | `bool` | `false` | `true` |
| `scheduler_schedule` | Cron schedule or interval for checking due instances. | `string` | `@every 15m` | `@every 5m` |
| `task_digest_enabled` | Register the task digest scan independently of workflow scheduling. | `bool` | `false` | `true` |
| `task_digest_schedule` | Cron schedule for task digests. | `string` | `0 0 8 * * *` | `0 0 9 * * *` |

### Environment overrides

| Setting | Purpose | Type | Default | Example |
|---|---|---|---|---|
| `CCF_WORKFLOW_CONFIG` | Select the YAML file; this is a main API setting, not a key inside the file. | `string` | `workflow.yaml` | `/etc/ccf/workflow.yaml` |
| `CCF_WORKFLOW_DUE_SOON_ENABLED` | Override `due_soon_enabled`. | `bool` | `false` | `true` |
| `CCF_WORKFLOW_DUE_SOON_SCHEDULE` | Override `due_soon_schedule`. | `string` | `0 0 8 * * *` | `0 0 9 * * *` |
| `CCF_WORKFLOW_GRACE_PERIOD_DAYS` | Override `grace_period_days`, in calendar days. | `int` | `7` | `14` |
| `CCF_WORKFLOW_OVERDUE_CHECK_ENABLED` | Override `overdue_check_enabled`. | `bool` | `true` | `true` |
| `CCF_WORKFLOW_SCHEDULER_ENABLED` | Override `scheduler_enabled`. | `bool` | `false` | `true` |
| `CCF_WORKFLOW_SCHEDULER_SCHEDULE` | Override `scheduler_schedule`. | `string` | `@every 15m` | `@every 5m` |
| `CCF_WORKFLOW_TASK_DIGEST_ENABLED` | Override `task_digest_enabled`. | `bool` | `false` | `true` |
| `CCF_WORKFLOW_TASK_DIGEST_SCHEDULE` | Override `task_digest_schedule`. | `string` | `0 0 8 * * *` | `0 0 9 * * *` |

The three schedule settings accept **six fields**: `second minute hour day-of-month month day-of-week`, or descriptors such as `@daily`, `@weekly` and `@every 15m`. Interval units include `s`, `m` and `h`. Five-field Unix cron is invalid. Cron uses the process's local time zone unless the expression specifies one, for example `CRON_TZ=UTC 0 0 8 * * *`. The `08:00` defaults represent UTC only when the process uses UTC.

The shipped sample `workflow.yaml` sets all three enable flags to `true` and sets reminder and digest schedules to `0 * * * * *` (every minute). Those are sample-file choices, not code defaults. For daily notifications, save this as `/etc/ccf/workflow.yaml`:

```yaml
scheduler_enabled: true
scheduler_schedule: "@every 15m"
grace_period_days: 7
overdue_check_enabled: true
due_soon_enabled: true
due_soon_schedule: "CRON_TZ=UTC 0 0 8 * * *"
task_digest_enabled: true
task_digest_schedule: "CRON_TZ=UTC 0 0 8 * * *"
```

Select that file in the API environment:

```sh
export CCF_WORKFLOW_CONFIG=/etc/ccf/workflow.yaml
```

Keep the `scheduler`, `workflow`, `email`, `digest` and notification delivery queues available. Due-soon scans run on `email`; digest scans run on `digest`. The scheduler flag does not enable the worker, reminder scan or digest scan. Periodic jobs do not run immediately at startup.

## Use it

### Create a definition and dependent steps

The shell examples require `curl` and `jq`. Set `TOKEN` to your authenticated user's token, `SSP_ID` to an existing SSP UUID and `USER_ID` to that same user's UUID. Use fictional test data in a test installation. Set your API origin:

```sh
export API_URL=https://api.example.com
```

Create a definition. Create responses use a `data` envelope; save each returned UUID:

```sh
DEF_ID=$(curl --fail-with-body -X POST "$API_URL/api/workflows/definitions" \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"name":"Fictional access review","version":"1.0","suggested-cadence":"monthly","grace-period-days":7}' | jq -er '.data.id')
```

Create a collection step that requires one `document` evidence submission:

```sh
COLLECT_ID=$(curl --fail-with-body -X POST "$API_URL/api/workflows/steps" \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d "{\"workflow-definition-id\":\"$DEF_ID\",\"name\":\"Collect review\",\"responsible-role\":\"reviewer\",\"estimated-duration\":30,\"grace-period-days\":3,\"evidence-required\":[{\"type\":\"document\",\"description\":\"Review record\",\"required\":true}]}" | jq -er '.data.id')
```

Create an approval step with a dependency on the collection **definition** ID:

```sh
APPROVE_ID=$(curl --fail-with-body -X POST "$API_URL/api/workflows/steps" \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d "{\"workflow-definition-id\":\"$DEF_ID\",\"name\":\"Approve review\",\"responsible-role\":\"reviewer\",\"grace-period-days\":4,\"depends-on\":[\"$COLLECT_ID\"]}" | jq -er '.data.id')
```

Dependencies must belong to the same definition and cannot form a cycle. Request field names use hyphens; response model fields can use underscores, such as `workflow_step_definition_id`.

### Bind the SSP and assign the role

Create an active instance. Use `monthly` in production, or a short cron cadence in a test installation to observe a scheduled run:

```sh
INSTANCE_ID=$(curl --fail-with-body -X POST "$API_URL/api/workflows/instances" \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d "{\"workflow-definition-id\":\"$DEF_ID\",\"name\":\"Fictional SSP review\",\"system-id\":\"$SSP_ID\",\"cadence\":\"monthly\",\"is-active\":true,\"grace-period-days\":7}" | jq -er '.data.id')
```

Assign the workflow role `reviewer` before starting. This role name is local to the workflow; it is separate from API authorization roles such as `contributor`. Assignment types are `user`, `group` and `email`; a `user` assignment uses the registered user's UUID:

```sh
curl --fail-with-body -X POST "$API_URL/api/workflows/role-assignments" \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d "{\"workflow-instance-id\":\"$INSTANCE_ID\",\"role-name\":\"reviewer\",\"assigned-to-type\":\"user\",\"assigned-to-id\":\"$USER_ID\",\"is-active\":true}"
```

### Start and complete a run

Start a manual execution to test the lifecycle without waiting for the next scheduled occurrence. The instance must be active. A successful `201` response confirms creation and enqueueing, not task initialization:

```sh
EXEC_ID=$(curl --fail-with-body -X POST "$API_URL/api/workflows/executions" \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d "{\"workflow-instance-id\":\"$INSTANCE_ID\",\"triggered-by\":\"manual\",\"triggered-by-id\":\"$USER_ID\"}" | jq -er '.data.id')
```

Wait for the worker, then list this execution's tasks. Repeat this read until the collection task is `pending` and the approval task is `blocked`:

```sh
curl --fail-with-body "$API_URL/api/workflows/step-executions?workflow_execution_id=$EXEC_ID" \
  -H "Authorization: Bearer $TOKEN"
```

Match `workflow_step_definition_id` to `COLLECT_ID` and `APPROVE_ID`. Save the returned **step execution** IDs as `COLLECT_TASK_ID` and `APPROVE_TASK_ID`. With the assigned user's token, start collection:

```sh
curl --fail-with-body -X PUT "$API_URL/api/workflows/step-executions/$COLLECT_TASK_ID/transition" \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"status":"in_progress"}'
```

Complete it with evidence matching the required `document` type:

```sh
curl --fail-with-body -X PUT "$API_URL/api/workflows/step-executions/$COLLECT_TASK_ID/transition" \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"status":"completed","notes":"Fictional review collected","evidence":[{"evidence-type":"document","name":"Fictional review record","description":"Reviewed fictional test accounts","file-content":"UmV2aWV3IGNvbXBsZXRlLgo=","media-type":"text/plain"}]}'
```

Required evidence is checked on completion: submit at least one item per required type. A `screenshot` submission with a media type must use a supported image type (`image/png`, `image/jpg`, `image/jpeg`, `image/gif` or `image/webp`). File content is base64 encoded. Read `GET /api/workflows/step-executions/{id}/evidence-requirements` for the task's requirements. Transition requests use the authenticated actor; supplying `user-id` or `user-type` does not impersonate another assignee.

Collection completion makes approval actionable. Send the same `in_progress` transition to `$APPROVE_TASK_ID`, then complete it (this step has no required evidence):

```sh
curl --fail-with-body -X PUT "$API_URL/api/workflows/step-executions/$APPROVE_TASK_ID/transition" \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"status":"completed","notes":"Fictional review approved"}'
```

Read execution status and confirm `completed` with two completed steps:

```sh
curl --fail-with-body "$API_URL/api/workflows/executions/$EXEC_ID/status" \
  -H "Authorization: Bearer $TOKEN"
```

### Verify scheduling and notifications

For a scheduled run, inspect `GET /api/workflows/instances/{id}` for `is_active`, `next-scheduled-at` and `last-executed-at`. After the next occurrence and scheduler tick, list executions:

```sh
curl --fail-with-body "$API_URL/api/workflows/executions?workflow_instance_id=$INSTANCE_ID" \
  -H "Authorization: Bearer $TOKEN"
```

Look for `triggered_by: "scheduled"`, a `period_label` and `due_date`, then inspect that execution's `/status` and tasks. API logs include `Processing workflow scheduler job`, `Found due workflow instances` and `Scheduled workflow execution`. A manual run alone does not verify scheduling.

Configure personal `taskAvailable` and `taskDailyDigest` subscriptions through [Notifications](./notifications.md#set-personal-subscriptions). Subscription updates replace the full preference set, so preserve your other choices.

| Behavior to verify | Preconditions and expected result |
|---|---|
| Available task | A pending task assigned to a user with `taskAvailable` enabled triggers `workflow_task_assigned`; unblocking a dependent task also triggers assignment notification processing. |
| Due-soon reminder | With `due_soon_enabled`, the scan selects `pending` or `in_progress` tasks due between now and seven days from now, assigned directly to a user. The `taskAvailable` subscription gates `workflow_task_due_soon` delivery. Blocked, overdue, group-assigned and email-assigned tasks are excluded from this scan. |
| Task digest | With `task_digest_enabled` and `taskDailyDigest`, the checker queues a per-user `workflow_task_digest`. It includes directly user-assigned `pending`, `in_progress` and `overdue` tasks, separated into pending and overdue groups. Users with no matching tasks receive no digest. |
| Execution failure | Configure a `workflow_execution_failed` system destination. In a disposable test execution, `PUT /api/workflows/step-executions/{id}/fail` with `{"reason":"Fictional delivery test"}` fails a step and skips its dependents. When all tasks are terminal and any failed, the execution fails and queues a failure notification. Other independent tasks may still need to finish. Expired overdue execution grace also queues failure notifications. |

With an administrator token, read `GET /api/admin/notifications/health` for worker, provider, queue and schedule health. Read `GET /api/admin/notifications/workflow/diagnostics` for workflow-family configuration, subscription counts and schedules. The aliases `task_available`, `task_daily_digest` and `workflow_execution_failed` also select this family; job kinds such as `workflow_task_due_soon` are not diagnostic names. Check delivery jobs, not only application logs:

```sh
curl --fail-with-body "$API_URL/api/admin/notifications/jobs?notificationKind=workflow_task_due_soon&limit=20" \
  -H "Authorization: Bearer $TOKEN"
```

Repeat with `notificationKind=workflow_task_digest` or `workflow_execution_failed`. Follow a job's ID with `GET /api/admin/notifications/jobs/{id}` to inspect state and errors. The [notification test endpoint](./notifications.md#send-a-test-notification) isolates transport delivery from workflow scheduling.

## Troubleshooting

| Symptom | Check and action |
|---|---|
| No scheduled executions | Check `CCF_WORKER_ENABLED`, `scheduler_enabled`, an active instance, a nonempty cadence and `next-scheduled-at`. Wait for the next periodic tick; jobs do not run on startup. Check the `scheduler` and `workflow` queues and API logs. |
| Invalid cron or YAML disables scheduling | The loader validates enabled schedules with a seconds field and rejects negative global grace. A load error logs `Failed to load workflow config, scheduler will be disabled` and replaces the workflow configuration with a disabled configuration, also disabling reminders and digests. Fix the file or environment values and restart. Missing files alone use defaults. |
| Execution stays `pending` with no tasks | Initialization is asynchronous. Check workers, the `workflow` queue and logs for `execute_workflow` initialization errors. |
| Unassigned roles or `403` on completion | Read `GET /api/workflows/role-assignments?workflow_instance_id={id}` and check the exact role name, active flag and assignee identity. Check both API permission and workflow assignment. Existing task assignees can require explicit reassignment: `PUT /api/workflows/step-executions/{id}/reassign` with `{"assigned-to-type":"user","assigned-to-id":"USER_UUID","reason":"Correct reviewer"}` requires `step-execution:update`. |
| Task stays `blocked` | Read its definition's dependencies with `GET /api/workflows/steps/{id}/dependencies` and the execution's task statuses. Finish prerequisites; blocked tasks cannot transition directly to completed. Failed prerequisites skip dependent tasks. |
| Completion rejects evidence | Compare `evidence-type` with every required `type`, include base64 file content where needed and check screenshot media types. Inspect the response and API logs for the missing type or storage error. |
| No overdue transition | Both `scheduler_enabled` and `overdue_check_enabled` must be true. Check `due_date`, the resolved grace and scheduler logs. Reminders and digests alone do not perform overdue checks. |
| No reminders or digest | Check the independent enable flags and schedules, direct user assignment, task status, due date, subscriptions and provider setup. A blocked task or empty task list produces no matching message. |
| Delivery stays queued or fails | Inspect notification health, jobs and per-kind diagnostics. Check workers, delivery queues, enabled providers, SMTP/SES or Slack connectivity, user subscriptions and system failure destinations. Enqueueing is not proof of delivery. |

## Related

- released UI `2.12` workflow guide
- [Notifications](./notifications.md)
- [Authentication and permissions](./authentication-and-permissions.md)
- [Configuration reference](./configuration.md)
- [Evidence artifacts and Rego playback](./evidence-and-playback.md)
- [API overview](./index.md)
- [Workflow configuration source at v0.21.1](https://github.com/compliance-framework/api/blob/v0.21.1/internal/config/workflow.go)
