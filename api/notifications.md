---
title: Notifications
description: Configure SMTP, Amazon SES and Slack notifications in API 0.21.1, set destinations and test delivery.
sidebar_position: 4
---

# Notifications

As an operator, you can configure the Continuous Compliance Framework (CCF) API to send email and Slack notifications. This guide covers API `v0.21.1` and the routes you use to select recipients and check delivery.

## How it works

The API supports two delivery channels: `email` and `slack`. Email uses SMTP or Amazon Simple Email Service (SES), selected in `email.yaml`. SMTP and SES are transports for the same `email` channel; use `email`, rather than `smtp` or `ses`, in API requests. SendGrid and other channels are not implemented in this release.

Enabling a provider makes it available for delivery. You also need recipients: system notification destinations or user subscriptions. Background workers process queued sends. Evidence digests follow `CCF_DIGEST_SCHEDULE` (`@weekly` by default) when `CCF_DIGEST_ENABLED` is `true` (the default).

## Before you start

- Run API `v0.21.1` with its database migrations applied. Keep `CCF_WORKER_ENABLED=true` (the default) to process queued notifications.
- Prepare SMTP credentials, an AWS identity for SES, or a Slack app installed in your workspace. The API needs network access to your provider.
- Use an authenticated user token for the routes below. `/api/admin/notifications` routes require permission to manage the `Admin` resource; the bundled Cedar `admin` role grants it. User subscriptions require read or update access to your own user resource. See [Authentication and permissions](./authentication-and-permissions.md).
- Set `CCF_WEB_BASE_URL` to your UI address so notification links lead to your installation.

The examples use fictional addresses and credentials. Replace them with your deployment's values and keep credential files outside version control.

## Enable it

### Email through SMTP

Save this as `/etc/ccf/email.yaml`, replacing the host, sender and credentials:

```yaml
enabled: true
provider: smtp
providers:
  smtp:
    name: SMTP
    enabled: true
    host: smtp.example.com
    port: 587
    username: ccf-notifications
    password: REPLACE_WITH_SMTP_PASSWORD
    from: notifications@example.com
    from_name: CCF notifications
    use_tls: true
    use_ssl: false
```

`use_tls: true` upgrades the connection with STARTTLS. For a server that requires implicit TLS, use its implicit TLS port (commonly `465`), set `use_ssl: true` and set `use_tls: false`. The API checks the connection and credentials when it initializes the SMTP provider.

Point the API at the file and restart it with these environment variables:

```sh
export CCF_EMAIL_CONFIG=/etc/ccf/email.yaml
```

### Email through Amazon SES

To use SES instead of SMTP, save this alternative as `/etc/ccf/email.yaml`:

```yaml
enabled: true
provider: ses
providers:
  ses:
    name: AWS SES
    enabled: true
    region: us-east-1
    access_key_id: ""
    secret_access_key: ""
    session_token: ""
    from: notifications@example.com
    from_name: CCF notifications
```

Use an SES sender identity verified in the configured region. Your AWS identity needs `ses:SendEmail` for delivery and `ses:GetAccount` for the provider's startup health check. In the SES sandbox, verify recipients too, or obtain production access.

Empty credential fields use the AWS SDK's default credential chain, including workload identity. Alternatively, populate the credential fields through the corresponding environment variables:

```sh
export CCF_EMAIL_PROVIDERS_SES_ACCESS_KEY_ID=REPLACE_WITH_ACCESS_KEY_ID
```

```sh
export CCF_EMAIL_PROVIDERS_SES_SECRET_ACCESS_KEY=REPLACE_WITH_SECRET_ACCESS_KEY
```

```sh
export CCF_EMAIL_PROVIDERS_SES_SESSION_TOKEN=REPLACE_WITH_SESSION_TOKEN
```

The session token is needed only for temporary credentials. Keep these keys present in the YAML when using overrides: the email loader reads the file before applying environment values. Both the top-level `enabled` and the selected provider's `enabled` must be `true`. Set `CCF_EMAIL_CONFIG` as in the SMTP example, then restart the API.

### Slack

Create and install a Slack app with a bot token that can post messages (`chat:write`). Invite the bot to the channels that should receive notifications. Save this as `/etc/ccf/slack.yaml`:

```yaml
enabled: true
token: REPLACE_WITH_SLACK_BOT_TOKEN
```

Point the API at the file and restart it:

```sh
export CCF_SLACK_CONFIG=/etc/ccf/slack.yaml
```

You can override the file's token with `CCF_SLACK_TOKEN`. Use Slack channel IDs, such as `C0123456789`, as system destinations. The legacy `digest_channel` setting is retained for a one-time migration into stored system destinations; manage new destinations through the routes below.

For personal Slack notifications, also configure Sign in with Slack so users can link their CCF and Slack accounts. Add the app's client credentials and registered redirect URL to the same file:

```yaml
enabled: true
token: REPLACE_WITH_SLACK_BOT_TOKEN
client_id: REPLACE_WITH_SLACK_CLIENT_ID
client_secret: REPLACE_WITH_SLACK_CLIENT_SECRET
redirect_url: https://api.example.com/api/auth/slack/link/callback
```

Register that exact callback URL with Slack. Account linking requests the `openid`, `profile` and `email` scopes. It is separate from installing the bot for channel delivery.

## Use it

### Check providers and configure system destinations

Set `API_URL` to your API origin and `TOKEN` to an authenticated administrator's bearer token in your shell. For example:

```sh
export API_URL=https://api.example.com
```

List providers and check their `enabled`, `error` and metadata fields:

```sh
curl --fail-with-body "$API_URL/api/admin/notifications/providers" \
  -H "Authorization: Bearer $TOKEN"
```

List system notification names and their `configuredDestinations`:

```sh
curl --fail-with-body "$API_URL/api/admin/notifications" \
  -H "Authorization: Bearer $TOKEN"
```

The system notification list includes `evidence_digest` and `workflow_execution_failed`. To send evidence digests to a Slack channel, add a destination:

```sh
curl --fail-with-body -X POST "$API_URL/api/admin/notifications/evidence_digest/destinations" \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"providerType":"slack","destinationTarget":"C0123456789"}'
```

To send workflow failure notifications by email, add an email destination:

```sh
curl --fail-with-body -X POST "$API_URL/api/admin/notifications/workflow_execution_failed/destinations" \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"providerType":"email","destinationTarget":"ccf-operators@example.com"}'
```

Remove a destination with `DELETE` on the same route and the same JSON body. Adding a duplicate returns `409`; deleting a destination that does not exist returns `404`.

### Send a test notification

A test send queues a message to an explicit target without waiting for a scheduled event:

```sh
curl --fail-with-body -X POST "$API_URL/api/admin/notifications/test" \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"providerType":"email","destinationTarget":"ccf-operators@example.com","mode":"enqueue"}'
```

For Slack, set `providerType` to `slack` and `destinationTarget` to the channel ID. A `202 Accepted` response contains `data.jobIds`; it confirms enqueueing, not delivery. Inspect those jobs through the troubleshooting routes below.

### Set personal subscriptions

With your own user token, read `GET /api/users/me/subscriptions` and update preferences with `PUT`:

```sh
curl --fail-with-body -X PUT "$API_URL/api/users/me/subscriptions" \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"notifications":{"evidenceDigest":["email","slack"],"taskAvailable":["email"],"taskDailyDigest":[],"riskNotifications":["email"]}}'
```

The four keys shown are the supported subscription types. This request replaces your subscription set; include every preference you want to keep. An empty channel list disables that type. Email goes to your account's email address. Personal Slack delivery requires a linked Slack account.

Account linking uses these routes:

| Method and route | Purpose |
|---|---|
| `GET /api/auth/slack/link/start` | Start linking in a browser with an authenticated CCF session; redirects to Slack. |
| `GET /api/auth/slack/link/callback` | Slack's callback; verifies state and records the link. |
| `GET /api/auth/slack/link/status` | Read your link status with authentication. |
| `DELETE /api/auth/slack/link` | Remove your link with authentication. |

## Troubleshooting

Use an administrator token for these routes:

| Method and route | What you can inspect |
|---|---|
| `GET /api/admin/notifications/health` | Provider, worker, queue, subscriber, destination and schedule health. |
| `GET /api/admin/notifications/jobs?provider=email&limit=20` | Recent notification jobs; filter by `provider`, `queue`, `notificationKind`, `state` or `since`, and paginate with `cursor`. |
| `GET /api/admin/notifications/jobs/{id}` | A queued job's state and errors, using an ID from the test response. |
| `GET /api/admin/notifications/{notificationName}/diagnostics` | Diagnostics for a notification name or family, for example `evidence_digest`. |

- **Provider disabled:** check the selected file path, YAML syntax and enable flags. Review the API startup logs for configuration loading errors, then restart after correcting the file.
- **Jobs stay queued:** check `CCF_WORKER_ENABLED`, database connectivity and queue health. A successful enqueue does not prove a worker processed the job.
- **SMTP initialization fails:** check the host, port, TLS mode, credentials and network access. The server's certificate must be trusted by the API.
- **SES initialization or delivery fails:** check the region, AWS credentials, `ses:GetAccount` and `ses:SendEmail` permissions, verified sender and sandbox recipient restrictions.
- **Slack sends fail:** check the bot token, app scopes and channel membership. Personal delivery also needs the user's Slack link and subscription.
- **Provider enabled but no notifications arrive:** check destinations, subscriptions and the relevant schedule. Non-admin clients can read enabled channel status from `GET /api/notifications/providers` with notification read permission; that response omits admin configuration details.

## Related

- [Configuration reference](./configuration.md)
- [Authentication and permissions](./authentication-and-permissions.md)
- [API overview](./index.md)
- [Email configuration at API v0.21.1](https://github.com/compliance-framework/api/blob/v0.21.1/internal/config/email.go)
- [Slack configuration at API v0.21.1](https://github.com/compliance-framework/api/blob/v0.21.1/internal/config/slack.go)
- [Notification routes at API v0.21.1](https://github.com/compliance-framework/api/blob/v0.21.1/internal/api/handler/notifications.go)
