---
title: Authentication and permissions
description: How users and agents sign in to the API, and how the API decides what each of them can do.
sidebar_position: 3
---

# Authentication and permissions

The Continuous Compliance Framework (CCF) API checks who's calling (authentication) and what they may do (authorization). This page is for **operators** who set up sign-in and roles, and **developers** who call the API.

## How it works

### Authentication

Every caller gets a JSON Web Token (JWT), signed with the API's RSA key, and sends it with each request.

| Caller | How it gets a token | How long the token lasts |
|---|---|---|
| A user in the UI | `POST /api/auth/login` with an email and password, or single sign-on (SSO). The API sets the token as the `ccf_auth_token` cookie. | 24 hours |
| A user or script calling the API | `POST /api/auth/token`, the OAuth2 password flow, with `username` (the email) and `password` as form fields. | 24 hours |
| A CCF Agent | `POST /api/auth/agent/token` with its service account's client ID and secret, as HTTP basic auth or as the `client_id` and `client_secret` form fields. | 24 hours |

Send the token as `Authorization: Bearer <token>`. The UI uses the cookie instead.

To verify tokens yourself, get the API's public key from `GET /api/auth/publickey` (a JSON Web Key) or `GET /api/auth/publickey.pub` (PEM). The signing keys come from `CCF_JWT_PRIVATE_KEY` and `CCF_JWT_PUBLIC_KEY`. Without them the API makes a new key pair at each start, and every token stops working when it restarts.

Configure OIDC or OAuth providers in `sso.yaml` (`CCF_SSO_CONFIG`). See [Single sign-on](./sso.md) for complete examples, account linking and login group requirements.

The endpoints for agent heartbeats, evidence and Rego playback also take requests without a token, unless you set `CCF_STRICT_DISABLE_PUBLIC_AGENT_ENDPOINTS=true`. The authorization engine still decides whether such a request is allowed.

### Authorization

Every request goes through the authorization engine you pick with `CCF_AUTHZ_DRIVER`:

| Driver | What it allows |
|---|---|
| `builtin` (default) | Every signed-in user can do everything except administration and agent management, which need admin access. Users who sign in with a password always have admin access. SSO users have it when every `required_admin_groups` entry is present in the union of their latest SSO link's mapped groups and native CCF memberships, or when the provider lists none. Agents may only register, send evidence and heartbeats, and sync their configuration. |
| `cedar` | Each user, group and agent gets a role from the role assignment file, and the bundled role policies decide. Anyone without a role is denied everything. |
| `authzen` | Every decision goes to your own policy decision point, which implements the AuthZen API, at `CCF_AUTHZ_ENDPOINT`. |

When the decision point can't be reached, `CCF_AUTHZ_FAIL_MODE` decides: `closed` (the default) denies the request, `open` allows it.

## Before you start

- Users who sign in with a password need an account: `api users add` creates one.
- For roles, you need the `cedar` driver and a role assignment file.

## Enable roles

1. Write a role assignment file (see [Assign roles](#assign-roles)).
2. Set the driver and the file's path, then restart the API:

   ```sh
   CCF_AUTHZ_DRIVER=cedar
   CCF_AUTHZ_ROLE_ASSIGNMENTS=/etc/ccf/authz-roles.yaml
   ```

`CCF_AUTHZ_ROLE_ASSIGNMENTS` defaults to `authz-roles.yaml` in the API's working directory. Without the file, only agents are authorized. An unknown role name in the file stops the API at startup.

## Roles

The `cedar` driver has these roles. "Read everything" covers every resource, agent configurations included. An agent's configuration overlay is redacted for anyone who can't also configure that agent.

| Role | What it can do |
|---|---|
| `admin` | Everything. |
| `contributor` | Read everything. Create, change and delete OSCAL documents, evidence, risks, POA&M items, dashboards and filters, and workflows. Publish and subscribe to SSP export offerings. No administration. |
| `auditor` | Read everything. Add evidence, create, update and promote risks, and create and update POA&M items. |
| `viewer` | Read everything. No changes. |
| `ssp-subscriber` | For a downstream system: update its own SSP, and read and subscribe to published SSP export offerings. It can't read anything else. |
| `agent` | For agents: send evidence, heartbeats and policy artifacts, register, sync their own configuration, and update their plugins' risk and subject templates. |

Every role, `agent` included, can run Rego playback.

## Assign roles

The role assignment file maps users, groups and agents to one role each. Names are matched without case.

```yaml
# Users, by the email they sign in with.
users:
  alice@example.com: admin
  bob@example.com: viewer

# Groups, by name.
groups:
  ccf-admins: admin
  ccf-users: contributor

# The role of every authenticated agent.
agents: agent

# The role of requests without a token. Leave it out to deny them (recommended).
# anonymous: viewer
```

A user's roles are their direct role plus the role of each native CCF group they belong to. SSO maps provider claims into native memberships at login; unmapped external groups do not reach Cedar. Manual memberships also apply. Group membership is resolved from the database for each request, rather than stored in the JWT. `required_admin_groups` applies to builtin authorization and does not assign a Cedar role. See [SSO groups and authorization](./sso.md#groups-and-authorization).

## Write your own policies

To go beyond the bundled roles without changing CCF, put your own Cedar policy files (`*.cedar`) in a directory and set `CCF_AUTHZ_CEDAR_POLICY_DIR` to it. The `cedar` driver adds them to the bundled role policies. A `forbid` policy overrides any `permit`, so your files can both grant more and take away. The entities are in the `CCF::` namespace, for example `CCF::User`, `CCF::Role` and `CCF::Action::"read"`. To get the schema, run:

```sh
api authz export --format cedar
```

## Local-account password recovery

Password recovery applies to accounts whose `auth_method` is empty or `password`. Recover SSO credentials at your identity provider; an account linked to SSO cannot use this local reset flow.

1. Configure email delivery using [Notifications](./notifications.md). Set `CCF_WEB_BASE_URL` to the UI address used in reset links, independently of SSO's `base_url`.
2. Request a reset with `POST /api/auth/forgot-password` and JSON `{"email":"operator@example.com"}`. A successful request returns `200` with the same generic message for unknown emails, SSO accounts and disabled email. This response does not prove that an email was sent. Check delivery logs and the worker when email is queued.
3. Follow the email link to `CCF_WEB_BASE_URL/auth/password-reset?token=...`. The signed token expires after **15 minutes**. Submit `POST /api/auth/password-reset` with JSON fields `token` (the emailed token) and `password` (your new password). Success returns `200`; an invalid or expired token returns `401`. Then sign in with the new password.

The reset token uses the API's JWT keys. Keep those keys stable so a restart does not invalidate outstanding reset links.

## Troubleshooting

- **Every request gets `401 Unauthorized` after a restart:** the API generated a new key pair. Set `CCF_JWT_PRIVATE_KEY` and `CCF_JWT_PUBLIC_KEY` so the keys survive restarts.
- **Every user is denied with the `cedar` driver:** check that the role assignment file exists at `CCF_AUTHZ_ROLE_ASSIGNMENTS` and assigns each user a role. For SSO group roles, check native memberships and the provider mapping; sign-in alone does not grant a role.
- **The sign-in cookie isn't kept over plain HTTP:** the cookie is `Secure` unless `CCF_ENVIRONMENT` is `local` or `development`.

## Related

- [Configuration reference](./configuration.md): every `CCF_AUTHZ_*`, `CCF_JWT_*` and `CCF_SSO_CONFIG` setting.
- [Single sign-on](./sso.md)
- [Notifications](./notifications.md): email delivery for password recovery.
- [API overview](./index.md)
