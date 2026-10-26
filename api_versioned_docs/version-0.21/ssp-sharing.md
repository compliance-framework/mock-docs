---
title: SSP sharing
description: Publish shared SSP capabilities, subscribe from another SSP, and verify responsibilities and drift.
---

# SSP sharing

Use the Continuous Compliance Framework (CCF) API to publish capabilities from one system security plan (SSP) and consume them in another. This guide is for developers integrating shared responsibility across SSPs, verified against API v0.21.1.

## How it works

The **upstream SSP** owns the implementation. A statement-level by-component export contains **provided capabilities** and **responsibilities** that consumers must discharge. A responsibility references a provided capability through its OSCAL `provided-uuid`, within the same export.

An **offering** curates those capabilities as items, each naming a control, statement, component and provided UUID. Publishing makes the offering discoverable through the cross-SSP catalog. This shares capabilities between SSPs; it does not download an OSCAL document file.

The **downstream SSP** subscribes to selected items. The API creates or reuses its implemented requirement, statement and by-component, plus a component representing the upstream system. It writes inherited entries, optional satisfied-responsibility entries and **leverage links** recording the offering version. Subscription does not create or require a leveraged authorization (an independently authored Authority to Operate record).

Satisfaction is `full` when every current upstream responsibility has a matching downstream satisfied entry, including when no responsibilities exist. Otherwise it is `partial`. Evidence posture is separate: `responsibilityPosture` reports `satisfied`, `not-satisfied` or `unknown` from attached evidence filters. Recording a responsibility as satisfied does not prove that its evidence passes.

## Before you start

- Run the API with its database migrations applied and authenticate with a bearer token. See [Authentication and permissions](./authentication-and-permissions.md).
- Prepare two SSPs. The downstream needs both `system-implementation` and `control-implementation`; subscription returns `422` if either is absent.
- In the upstream, prepare an implemented requirement, a statement and a by-component for the capability. Keep its requirement UUID, statement UUID, by-component UUID, component UUID, catalog control ID and exact OSCAL statement ID. The statement UUID in a URL differs from its OSCAL `statement-id` string in an offering item.
- Prepare an evidence filter for the downstream if you want responsibility posture. See [Dashboards and AI filter suggestions](./dashboard-suggestions.md) for filter examples.

Permissions follow these boundaries in the shipped authorization manifest:

| Operation | Permission | Shipped role behavior |
|---|---|---|
| Curate, publish, change status or edit allowed downstreams on SSP-nested offering routes | `ssp:export` on upstream | `contributor` has it. |
| List/read the cross-SSP offering catalog | `ssp-export-offering:read` | `contributor` and `ssp-subscriber` have it. |
| Subscribe | `ssp-export-offering:subscribe` plus `ssp:update` on downstream | Both roles have these actions. No upstream `ssp:read` check occurs. |
| Read leveraged controls, responsibility filters or shared-responsibility rollup | `ssp:read` on downstream | `contributor` has it; `ssp-subscriber` alone does not. |
| Re-attest a drifted link | `ssp:update` on downstream | Both roles have it. |
| Create upstream export/provided/responsibility records | `ssp:create` | `contributor` has it; `ssp-subscriber` does not. |
| Attach a responsibility to a filter | `filter:update` | `contributor` has it; `ssp-subscriber` does not. |

The shipped `ssp-subscriber` role has no wildcard read permission. Give a consuming integration separate downstream read access through your authorization policy if it must inspect posture. Do not grant broad upstream SSP access for that purpose. The shipped roles are type-level grants; an allow-list restricts subscription destinations, not the role's general SSP permissions.

## Enable it

No sharing-specific environment setting is required. Create and publish an offering to expose a capability.

An offering with **zero allowed-downstream rows permits any downstream** that passes subscription authorization. With one or more rows, only listed SSPs may subscribe. Removing the last row restores the unrestricted default. This is not a catalog visibility restriction: the flat catalog still lists published offerings. The by-control catalog can filter eligible offerings with `downstreamSspId`, and subscription enforces the allow-list itself.

## Use it

### Prepare the example

The commands use `curl` and `jq`. Set `API` to your API URL including `/api`. Supply tokens through your environment as `PUBLISHER_TOKEN`, `SUBSCRIBER_TOKEN` and `READER_TOKEN`. The publisher and reader in this example have contributor permissions; the subscriber has `ssp-subscriber`. Replace the sample identifiers below with your existing records, except the two new UUIDs for the provided capability and responsibility.

```sh
export API='https://ccf.example.test/api'
export UPSTREAM='11111111-1111-4111-8111-111111111111'
export DOWNSTREAM='22222222-2222-4222-8222-222222222222'
export REQUIREMENT='33333333-3333-4333-8333-333333333333'
export STATEMENT='44444444-4444-4444-8444-444444444444'
export BY_COMPONENT='55555555-5555-4555-8555-555555555555'
export COMPONENT='66666666-6666-4666-8666-666666666666'
export PROVIDED='77777777-7777-4777-8777-777777777777'
export RESPONSIBILITY='88888888-8888-4888-8888-888888888888'
export FILTER='99999999-9999-4999-8999-999999999999'
export CONTROL_ID='AC-2'
export STATEMENT_ID='ac-2_smt.a'
export SSPS="$API/oscal/system-security-plans"
export CATALOG="$API/oscal/ssp-export-offerings"
export EXPORT_PATH="$SSPS/$UPSTREAM/control-implementation/implemented-requirements/$REQUIREMENT/statements/$STATEMENT/by-components/$BY_COMPONENT/export"
```

### Create the upstream capability and offering

Create an export on the existing statement by-component. This payload uses OSCAL's kebab-case field names. If that by-component already has an export, use its `/provided` and `/responsibilities` subresources instead; creating a second export returns `409`.

```sh
curl --fail-with-body -X POST "$EXPORT_PATH" \
  -H "Authorization: Bearer $PUBLISHER_TOKEN" -H 'Content-Type: application/json' \
  -d "{\"provided\":[{\"uuid\":\"$PROVIDED\",\"description\":\"Fictional platform account management\"}],\"responsibilities\":[{\"uuid\":\"$RESPONSIBILITY\",\"provided-uuid\":\"$PROVIDED\",\"description\":\"Review application accounts\"}]}"
```

Create a draft offering. `title` is required; `description` is optional. These offering endpoints use camelCase rather than OSCAL field names.

```sh
OFFERING=$(curl --fail-with-body -X POST "$SSPS/$UPSTREAM/export-offerings" \
  -H "Authorization: Bearer $PUBLISHER_TOKEN" -H 'Content-Type: application/json' \
  -d '{"title":"Fictional platform account management"}' | jq -er '.data.id')
```

Add a statement-anchored item. All four fields are required. They must resolve to the same upstream implementation: provided capability → export → statement by-component → statement → requirement → upstream SSP. Control matching ignores case and stores canonical casing; statement IDs must match exactly.

```sh
ITEM=$(curl --fail-with-body -X POST "$SSPS/$UPSTREAM/export-offerings/$OFFERING/items" \
  -H "Authorization: Bearer $PUBLISHER_TOKEN" -H 'Content-Type: application/json' \
  -d "{\"controlId\":\"$CONTROL_ID\",\"statementId\":\"$STATEMENT_ID\",\"componentUuid\":\"$COMPONENT\",\"providedUuid\":\"$PROVIDED\"}" | jq -er '.data.id')
```

Restrict subscriptions before publishing. Add an existing downstream SSP with `POST`, inspect with `GET`, and remove a row with `DELETE .../allowed-downstreams/{downstreamSspId}`. Adding the same destination again leaves the existing row in place and still returns `201`.

```sh
curl --fail-with-body -X POST "$SSPS/$UPSTREAM/export-offerings/$OFFERING/allowed-downstreams" \
  -H "Authorization: Bearer $PUBLISHER_TOKEN" -H 'Content-Type: application/json' \
  -d "{\"downstreamSspId\":\"$DOWNSTREAM\"}"
```

Publish without a request body. Save the returned `version` and `contentHash` for later comparison.

```sh
curl --fail-with-body -X POST "$SSPS/$UPSTREAM/export-offerings/$OFFERING/publish" \
  -H "Authorization: Bearer $PUBLISHER_TOKEN"
```

### Discover and subscribe

List published offerings using the subscriber token. Items include resolved `responsibilities`, with `responsibilityUuid` and `description`, so you do not need to read the upstream SSP. List and by-control queries use `limit` (default `100`, maximum `1000`) and `offset` (default `0`); continue through pages as needed.

```sh
curl --fail-with-body "$CATALOG?limit=100&offset=0" \
  -H "Authorization: Bearer $SUBSCRIBER_TOKEN"
```

Read one offering with `GET $CATALOG/$OFFERING`, or find eligible items for one control:

```sh
curl --fail-with-body "$CATALOG/by-control/$CONTROL_ID?downstreamSspId=$DOWNSTREAM" \
  -H "Authorization: Bearer $SUBSCRIBER_TOKEN"
```

Subscribe to the item, recording that the downstream discharges the example responsibility. Only select responsibility UUIDs returned for that item's provided capability. The minimal subscription omits `satisfiedResponsibilityUuids`; the link then remains `partial` if responsibilities are outstanding.

```sh
LINK=$(curl --fail-with-body -X POST "$CATALOG/$OFFERING/subscribe" \
  -H "Authorization: Bearer $SUBSCRIBER_TOKEN" -H 'Content-Type: application/json' \
  -d "{\"downstreamSspId\":\"$DOWNSTREAM\",\"items\":[{\"itemId\":\"$ITEM\",\"satisfiedResponsibilityUuids\":[\"$RESPONSIBILITY\"]}]}" | jq -er '.data[0].id')
```

Success returns `201`, leverage links in `data`, and `meta.created` describing requirements, statements and by-components created or reused. Subscription is atomic. A repeated subscription for the same downstream and provided UUID returns `409`; it does not refresh an existing link.

### Attach evidence and inspect downstream posture

With a token that has `filter:update`, attach an existing filter to the inherited responsibility. The filter must belong to this downstream SSP or be global. A filter owned by another SSP is rejected. This minimal body does not add a control link.

```sh
curl --fail-with-body -X POST "$API/filters/$FILTER/responsibilities" \
  -H "Authorization: Bearer $READER_TOKEN" -H 'Content-Type: application/json' \
  -d "{\"responsibilityUuid\":\"$RESPONSIBILITY\",\"sspId\":\"$DOWNSTREAM\"}"
```

You can also supply `controlId` to link the filter to a control. The API resolves its catalog through the downstream SSP's profiles; ambiguous catalogs return `400`, and a missing control returns `404`. Attachment requires a responsibility inherited through a link that is not `revoked` or `superseded`; a `drifted` link qualifies. Duplicate attachments return `409`.

Read the downstream projection with downstream `ssp:read`. It returns `inheritedFrom`, `providedUuid`, `byComponentId`, `satisfaction`, `status`, `responsibilities`, `outstandingResponsibilities`, `responsibilityPosture` and, when present, `driftRiskId`.

```sh
curl --fail-with-body "$SSPS/$DOWNSTREAM/leveraged-controls" \
  -H "Authorization: Bearer $READER_TOKEN"
```

Inspect filter attachments, including filter names:

```sh
curl --fail-with-body "$SSPS/$DOWNSTREAM/responsibility-filters" \
  -H "Authorization: Bearer $READER_TOKEN"
```

Read the combined posture, optionally narrowed to a control:

```sh
curl --fail-with-body "$SSPS/$DOWNSTREAM/shared-responsibility?controlId=$CONTROL_ID" \
  -H "Authorization: Bearer $READER_TOKEN"
```

The rollup has `provides`, `inherits`, `satisfies` and `legacy` arrays. Its inherited satisfaction uses the same live projection as leveraged controls. On an upstream SSP, `provides[].offered` indicates whether a published offering references a provided capability. `legacy` identifies requirement-anchored shared-responsibility records, not ordinary requirement implementations. None of these downstream reads requires general read access to the upstream SSP.

### Republish and reconcile drift

Edit offering metadata with `PUT .../export-offerings/{offeringId}` using `title` and optional `description`. Edit items through their `POST`, `PUT` and `DELETE` routes. Then call the same `/publish` endpoint again.

Publishing recomputes a deterministic SHA-256 `contentHash` from title, description and sorted item tuples, including the live backing by-component implementation status. It increments `version` only when the hash changes. Reordering items or republishing unchanged content leaves the version unchanged, although publishing refreshes `publishedAt`. Metadata/item edits alone do not update the hash or version. Responsibility descriptions and satisfied selections are not hash inputs; satisfaction is recomputed live on reads.

When a new offering version exceeds a link's recorded `offeringVersion`, synchronization marks active links `drifted`. Deprecating or revoking a published offering also marks active links drifted, independently of its content hash. The API creates or reopens a deduplicated downstream `inherited-revoked` risk per link and enqueues drift notifications when a job enqueuer is available. Repeated triggers do not re-drift an already drifted link.

To retire an offering, send `PATCH .../export-offerings/{offeringId}/status` with `{"status":"deprecated"}` or `{"status":"revoked"}`. Only a published offering can transition this way. Both states disappear from the flat catalog, return `404` on flat reads/subscriptions, and cannot be published again. Existing inherited entries remain; revoking an offering drifts its links rather than changing their status to `revoked`.

Inspect the current offering through the upstream curator if it is no longer in the catalog. Review changes, outstanding responsibilities and evidence before acknowledging drift:

```sh
curl --fail-with-body -X POST "$SSPS/$DOWNSTREAM/leveraged-controls/$LINK/attest" \
  -H "Authorization: Bearer $SUBSCRIBER_TOKEN"
```

Re-attestation takes no body. It requires a drifted link, records the current offering version, recomputes satisfaction, sets the link `active`, records the actor/time and marks a matching non-closed drift risk `remediated`. It does not recreate the inherited tree or replace satisfied entries. In v0.21.1, it does **not** reject a deprecated or revoked offering, or recheck its allow-list. Treat it as an explicit acceptance decision, not proof that the capability is still available. Verify the projection and risk after the operation.

### Inspect lineage and inherited posture

Lineage is a read-only view of catalog/control links, evidence, risks and inherited capabilities. With a user token and `lineage:read`, list active catalog roots, optionally scoping metrics to a downstream SSP:

```sh
curl --fail-with-body -sS --get "$API/lineage/roots" \
  -H "Authorization: Bearer $READER_TOKEN" --data-urlencode "sspId=$DOWNSTREAM"
```

Use returned node `key` values rather than constructing identifiers. URL-encode the key as a single path segment (for example, with `jq -rn --arg key "$NODE_KEY" '$key|@uri'`). `GET /api/lineage/nodes/{encoded-key}/children` traverses children and accepts `page`/`limit`. Roots and children accept `sspId` and `componentId` metric scopes. For a control key, `/ssps` provides per-SSP posture and `/leverage?sspId={downstream-uuid}` returns offering origins, outstanding responsibilities, live responsibility posture and drift risk IDs. The leverage route ignores `componentId`; its `sspId` selects downstream rows. Invalid scope UUIDs or node keys return `400`, and unknown controls return `404` on leverage detail.

Inspect both `status` and `satisfaction` on links. A full but drifted link is not the same as current inherited credit. Global compliance totals with SSPs count control/SSP pairs; a control tracked by several plans can contribute several times. SSP-scoped totals count in-scope controls. Declared `planned` or `not-applicable` posture does not rewrite the raw compliance denominator. See this version's **API reference**, under Lineage, for node response fields.

## Troubleshooting

| Symptom | Verify and resolve |
|---|---|
| Offering missing or subscription returns `404` | Flat routes expose only `published` offerings. A curator can inspect `GET .../system-security-plans/{upstream}/export-offerings/{offering}` and publish a draft. Deprecated/revoked offerings cannot be republished; curate a replacement. |
| `403` when subscribing | Check offering `subscribe`, downstream `ssp:update`, and the offering's allowed-downstream rows. A nonempty list must include the downstream; an empty list permits any authorized downstream. |
| Subscriber can subscribe but cannot read posture | The shipped `ssp-subscriber` role lacks `ssp:read`. Add downstream read permission through your policy or use an authorized reader. Filter attachment separately requires `filter:update`. |
| Item creation/update returns `400` | Check the full upstream ownership chain. `statementId` is the exact OSCAL identifier, not the statement row UUID. Component, provided UUID and control must describe that same statement implementation. |
| Subscription returns `422` | Re-curate legacy items without statement IDs. Also verify the downstream has system and control implementations. |
| Responsibility selection or attachment returns `400` | Use the responsibility UUID for the selected provided capability in the same export. For attachment, the downstream must inherit it and the filter must be global or owned by that SSP. |
| Subscription or attachment returns `409` | Inspect existing links/attachments rather than repeating creation. A downstream cannot subscribe twice to the same provided UUID. |
| Satisfaction is `full` but posture is `unknown` or `not-satisfied` | Satisfied entries and evidence posture measure different things. Inspect responsibility filters and matching evidence; no attached filter produces `unknown`. |
| Link remains drifted | Inspect `status`, `inheritedFrom.offeringVersion`, current offering version and `driftRiskId`. Review changes and explicitly re-attest. Attesting a non-drifted link returns `400`; a concurrent status change can return `409`. |

## Related

- [Authentication and permissions](./authentication-and-permissions.md)
- [Dashboards and AI filter suggestions](./dashboard-suggestions.md)
- [Risk and POA&M automation](./risk-and-poam-automation.md)
- [Notifications](./notifications.md)
- [OSCAL import and export](./oscal-import-export.md): document interchange without subscriptions.
- [API overview](./index.md)
