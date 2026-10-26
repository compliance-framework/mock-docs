---
title: Single sign-on
description: Configure OIDC and OAuth providers, group access and sign-in verification for the API.
sidebar_position: 5
---

# Single sign-on

As an operator, you can connect the Continuous Compliance Framework (CCF) API to your identity provider so users can sign in without a local password. This guide covers API `v0.21.1`.

## How it works

The API supports generic OpenID Connect (OIDC), Google OIDC and GitHub OAuth. `protocol` defaults to `oidc`; GitHub needs `oauth`. Generic OAuth is also supported when you supply its authorization, token and user-info endpoints.

| Route | Behavior |
|---|---|
| `GET /api/auth/sso/providers` | Returns a `data` array with each enabled provider's `name`, `displayName`, `enabled` and optional `iconUrl`. Disabled SSO returns an empty array. |
| `GET /api/auth/sso/:provider` | Sets the five-minute `oauth_state` cookie and redirects the browser to the provider. |
| `GET /api/auth/sso/callback/:provider` | Checks state, exchanges the code, loads identity and groups, then sets the 24-hour `ccf_auth_token` cookie. Redirects to the UI at `base_url/auth/sso/callback?provider=:provider`. |

The callback first reuses the link for the provider and external subject. Otherwise it links an existing account with the same email and changes its `auth_method` to `sso`, or creates an active account and a provider link. The handler does not add an email-verification check before linking: trust only providers whose email identities you control. Linking a local account makes local password recovery unavailable for that account.

### Groups and authorization

`group_mapping` translates external claim keys to CCF group names. For example, `groups:operators` matches a value in the OIDC `groups` array; `hd:example.com` and `email:operator@example.com` match string claims. Claim mapping is case insensitive. Unmapped claims grant no groups.

The callback requires **every** group in `required_login_groups`, for both existing and new users. Comparisons trim whitespace and ignore case. An empty list allows any identity from the provider and allows account creation without a mapped group. Native groups assigned manually do not bypass this login gate.

At startup, the API creates native groups and database mappings declared in enabled providers' `group_mapping`. At login, raw provider claims pass through those database mappings into native memberships with source `sso`. The sync removes lost SSO memberships and preserves manual memberships. Sync failures are logged but do not block login; previous memberships remain. Runtime database mappings affect native membership sync, while the login gate still uses the provider's configured mapping.

With `CCF_AUTHZ_DRIVER=builtin`, signed-in users can use ordinary resources. Administration and agent management require **all** `required_admin_groups`, satisfied by the union of the latest SSO link's mapped groups and native memberships. An empty admin list grants admin access to that provider's users; it does not mean no admins. Non-SSO accounts bypass this admin group check. Disabling SSO also skips provider admin group enforcement. Keep provider map keys equal to `name` so admin lookup finds the configuration.

With `cedar`, native groups feed group role assignments alongside direct user assignments. `required_admin_groups` does not grant a Cedar role. Map a CCF group to a role such as `admin` in the role assignment file. See [Authentication and permissions](./authentication-and-permissions.md#assign-roles).

## Before you start

- Run API `v0.21.1` with its database migrations applied. You need access to the API's configuration files and permission to restart it, plus access to register an application at your identity provider.
- Prepare HTTPS API and UI addresses. The examples use fictional `api.example.com`, `ccf.example.com`, identities and credentials. Replace them with your deployment values.
- Register the exact callback URL for each provider: `callback_url` plus `/` plus its `name`. For `workforce` below, register `https://api.example.com/api/auth/sso/callback/workforce`. Do not add a trailing slash to `callback_url`.
- Allow the API to reach discovery, token, signing-key and user-info endpoints. The browser must reach the provider's authorization endpoint and your API callback. Use a deployment where the UI can send the API authentication cookie.
- Decide who may sign in and who may administer CCF. For Cedar, prepare [role assignments](./authentication-and-permissions.md#assign-roles). For builtin, set a nonempty `required_admin_groups` list when you need to restrict admin access.

## Enable it

### Configuration settings

Save one of the complete configurations below as `/etc/ccf/sso.yaml`. Both the root `enabled` and each selected provider's `enabled` must be `true`. `providers` is a map, not a list. Keep its key and the provider `name` identical.

| Setting | Purpose | Type | Default | Example |
|---|---|---|---|---|
| `CCF_SSO_CONFIG` | Select the SSO YAML file. | `string` | `sso.yaml` | `/etc/ccf/sso.yaml` |
| `base_url` | UI address for success and error redirects. | `string` | Empty | `https://ccf.example.com` |
| `callback_url` | API callback prefix, before the provider name. | `string` | Empty | `https://api.example.com/api/auth/sso/callback` |
| `enabled` | Enable SSO globally. | `bool` | `false` | `true` |
| `providers` | Provider configurations keyed by provider identifier. | `map` | Empty | See examples below |

These keys live under `providers.<key>`:

| Setting | Purpose | Type | Default | Example |
|---|---|---|---|---|
| `auth_url` | OAuth authorization endpoint; OIDC uses discovery. | `string` | Empty | `https://github.com/login/oauth/authorize` |
| `client_id` | Application client ID; required for enabled providers. | `string` | Empty | `fictional-ccf-client` |
| `client_secret` | Application secret; required for enabled providers. | `string` | Empty | `REPLACE_WITH_CLIENT_SECRET` |
| `display_name` | Provider label returned by discovery. | `string` | Empty | `Workforce` |
| `email_url` | OAuth fallback endpoint when user info has no email. | `string` | Empty | `https://api.github.com/user/emails` |
| `enabled` | Enable this provider. | `bool` | `false` | `true` |
| `group_mapping` | Translate claim keys to CCF group names. | `map of lists` | Empty | `"groups:operators": [ccf-users]` |
| `icon_url` | Optional icon returned by provider discovery. | `string` | Empty | `https://ccf.example.com/sso-icon.svg` |
| `issuer_url` | Expected OIDC issuer and default discovery base. | `string` | Empty | `https://identity.example.com` |
| `name` | Provider route identifier and callback suffix. | `string` | Empty | `workforce` |
| `protocol` | Select `oidc` or `oauth`. | `string` | `oidc` | `oauth` |
| `provider` | Select `generic`, `google` or `github`; unknown values fall back to the selected protocol's generic implementation. | `string` | Generic implementation | `generic` |
| `required_admin_groups` | Groups all required by builtin for admin access. Empty grants admin access. | `list` | Empty | `[ccf-admins]` |
| `required_login_groups` | Groups all required before account lookup or creation. | `list` | Empty | `[ccf-users]` |
| `scopes` | Requested scopes; explicit lists replace defaults. | `list` | OIDC: `[openid, email, profile]`; OAuth: `[user:email]` | `[openid, email, profile, groups]` |
| `token_url` | OAuth token endpoint; OIDC uses discovery. | `string` | Empty | `https://github.com/login/oauth/access_token` |
| `user_info_url` | OAuth user-info endpoint; OIDC reads verified ID-token claims. | `string` | Empty | `https://api.github.com/user` |
| `well_known_url` | Optional full OIDC discovery URL, with endpoint rewriting described below. | `string` | Empty | `https://identity.internal.example.com/.well-known/openid-configuration` |

### Generic OIDC

```yaml
enabled: true
base_url: https://ccf.example.com
callback_url: https://api.example.com/api/auth/sso/callback
providers:
  workforce:
    name: workforce
    display_name: Workforce
    provider: generic
    protocol: oidc
    enabled: true
    client_id: fictional-ccf-client
    client_secret: REPLACE_WITH_WORKFORCE_SECRET
    issuer_url: https://identity.example.com
    scopes: [openid, email, profile, groups]
    required_login_groups: [ccf-users]
    required_admin_groups: [ccf-admins]
    group_mapping:
      "groups:operators": [ccf-users]
      "groups:administrators": [ccf-users, ccf-admins]
```

The provider must return an ID token with email and the claims you map. Without `well_known_url`, discovery uses `issuer_url` and the standard `/.well-known/openid-configuration` suffix.

For a separate internal discovery address, add `well_known_url` to the provider. Its path must end in `/.well-known/openid-configuration`. The discovery document's `issuer` must exactly equal `issuer_url`, which remains the issuer used for ID-token validation. The API derives an internal issuer base by removing the well-known suffix. It rewrites token, user-info and JWKS URLs that equal or begin with the public issuer plus `/` to that internal base. It leaves the browser authorization URL and endpoints on other bases unchanged. This is more than an alternate discovery fetch URL: ensure the internal base serves those rewritten endpoints.

### Google OIDC

Register `https://api.example.com/api/auth/sso/callback/google` at Google. This example maps the fictional Workspace hosted domain and one admin email:

```yaml
enabled: true
base_url: https://ccf.example.com
callback_url: https://api.example.com/api/auth/sso/callback
providers:
  google:
    name: google
    display_name: Google
    provider: google
    protocol: oidc
    enabled: true
    client_id: fictional-client.apps.googleusercontent.com
    client_secret: REPLACE_WITH_GOOGLE_SECRET
    issuer_url: https://accounts.google.com
    scopes: [openid, email, profile]
    required_login_groups: [ccf-users]
    required_admin_groups: [ccf-admins]
    group_mapping:
      "hd:example.com": [ccf-users]
      "email:operator@example.com": [ccf-admins]
```

The admin also needs the hosted-domain mapping to pass the login gate. For Workspace groups, add scope `https://www.googleapis.com/auth/admin.directory.group.readonly` and map `google-group:operators@example.com`. The Directory API must permit the user's token to read groups. Group-fetch failures do not fail sign-in directly, but missing groups can fail the login gate. The basic `openid`, `email` and `profile` scopes do not grant Directory group access.

### GitHub OAuth

Register `https://api.example.com/api/auth/sso/callback/github` in your GitHub OAuth application. Replace the fictional organization and team with your own:

```yaml
enabled: true
base_url: https://ccf.example.com
callback_url: https://api.example.com/api/auth/sso/callback
providers:
  github:
    name: github
    display_name: GitHub
    provider: github
    protocol: oauth
    enabled: true
    client_id: fictional-github-client
    client_secret: REPLACE_WITH_GITHUB_SECRET
    auth_url: https://github.com/login/oauth/authorize
    token_url: https://github.com/login/oauth/access_token
    user_info_url: https://api.github.com/user
    email_url: https://api.github.com/user/emails
    scopes: [read:user, user:email, read:org]
    required_login_groups: [ccf-users]
    required_admin_groups: [ccf-admins]
    group_mapping:
      "github-organization:fictional-ccf-lab": [ccf-users]
      "github-team:fictional-ccf-lab/platform": [ccf-users, ccf-admins]
```

GitHub adds organization and `organization/team-slug` keys to mapped profile claims. Ensure the OAuth grant can read the organization and team memberships you require. Failed organization or team fetches leave those groups absent. If the profile email is empty, the API reads `email_url`, preferring the primary email and otherwise the first returned email.

### Secrets and startup

The loader does **not** expand `${VAR_NAME}` in YAML. It reads environment overrides with prefix `CCF_SSO_`; nested key separators, dots and hyphens become underscores. Keep `client_secret` present in the YAML so unmarshalling discovers the key. For the generic example, inject the real secret into the API process with `CCF_SSO_PROVIDERS_WORKFORCE_CLIENT_SECRET`. Google and GitHub use `CCF_SSO_PROVIDERS_GOOGLE_CLIENT_SECRET` and `CCF_SSO_PROVIDERS_GITHUB_CLIENT_SECRET`. There is no separate `*_FILE` secret loader. Empty environment values do not override the YAML value.

For example, after your secret manager injects the override, set the file path and restart the API:

```sh
export CCF_SSO_CONFIG=/etc/ccf/sso.yaml
```

Keep the YAML and process environment outside version control and restrict access to them. Replace every fictional credential; placeholder strings pass the nonempty validation but cannot authenticate.

## Use it

1. Check provider discovery:

   ```sh
   curl --fail https://api.example.com/api/auth/sso/providers
   ```

   The `data` array should include your provider, for example `name: workforce`, `displayName: Workforce` and `enabled: true`. Discovery lists configured enabled providers even if initialization failed, so also verify sign-in.
2. In a browser, navigate to `https://api.example.com/api/auth/sso/workforce`. Authenticate with a test identity that has the mapped login group. Keep the same browser through the callback so it retains `oauth_state`.
3. Verify the callback sets `ccf_auth_token` and redirects to `https://ccf.example.com/auth/sso/callback?provider=workforce` without a login error. The cookie is HttpOnly; inspect its presence in browser developer tools rather than JavaScript.
4. In that browser's API session, request `GET /api/users/me` and confirm the returned email is your test identity. Check an operation allowed by the expected role, then confirm a user without the login group gets `missing_group`. With builtin, also test admin access separately using a user without `ccf-admins`.

These steps verify the API redirect flow; they do not require a particular UI sign-in button.

## Troubleshooting

- **Empty discovery array or SSO route returns `404`:** check `CCF_SSO_CONFIG`, readable YAML, root and provider `enabled` flags. Missing, malformed or invalid files disable SSO in the main configuration loader and produce a warning. Every enabled provider needs nonblank `client_id` and `client_secret`, even when root SSO is disabled.
- **Provider is listed but starting sign-in returns `404`:** check API logs for `Failed to initialize SSO provider`. OIDC discovery, issuer mismatch or an unsupported protocol can prevent initialization while the provider remains listed. Check network access and keep the map key equal to `name`.
- **`invalid_state`:** restart sign-in in the same browser within five minutes. Allow the `oauth_state` cookie. Use HTTPS in production; the cookie is Secure except when `CCF_ENVIRONMENT` is `local` or `development`. Parallel sign-in attempts can overwrite the state cookie.
- **Provider rejects the redirect or callback reports `no_code` or `token_exchange_failed`:** register the full callback including the provider name, check the client ID and secret, and avoid a trailing slash on `callback_url`. Provider errors are redirected to `base_url/login?error=...` after state validation.
- **`user_info_failed`:** check ID-token issuer, audience and signing keys for OIDC, or the OAuth user-info/email endpoints and scopes. With `well_known_url`, confirm the rewritten token and JWKS endpoints work.
- **`missing_group`:** check the provider's actual claims and `group_mapping`. You need every configured login group, not one of them. For Google/GitHub, check the optional membership API permissions and use the exact group key formats above.
- **Sign-in succeeds but access returns `403`:** login groups allow entry, not a Cedar role. Check native memberships and role assignments. With builtin, check every admin requirement and the latest SSO link. An empty admin list allows admin access.
- **`user_creation_failed` or `token_generation_failed`:** inspect API logs for account/link database errors or JWT signing errors. For group-sync errors, inspect native memberships too: sync failure preserves prior membership and still allows sign-in.

## Related

- [Authentication and permissions](./authentication-and-permissions.md)
- [Configuration reference](./configuration.md)
- [Notifications](./notifications.md): email setup for local-account password recovery.
- [API overview](./index.md)
