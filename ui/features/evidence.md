---
title: Evidence
description: Search evidence, inspect every detail tab, and save searches as evidence filters linked to controls.
sidebar_position: 2
---

# Evidence

Use evidence in the Continuous Compliance Framework (CCF) to review check results and link them to controls. This guide is for compliance and GRC users.

## How it works

Each evidence record has a result and labels that identify its source. Records in the same evidence stream share a stream identifier. New records become the stream's latest result; earlier records remain in its history.

You find evidence by searching its labels. An evidence filter saves a search and links it to the controls it supports. A system's compliance uses the evidence found by its controls' filters.

## Before you start

- The recordings use UI `v2.12.1` and API `v0.21.1` in a temporary environment with fictional data.
- To search evidence and read its details, you need `evidence:read`. To see evidence filters, you need `filter:read`.
- To create filters, use a role that can change filters, such as `contributor`. Creating a risk also requires `risk:create`. Re-submitting evidence requires `evidence:update` in the UI.
- Your API needs evidence to show results. Policy replay requires the artifacts stored with that evidence; media requires linked back matter resources.

## Use it

### Search evidence

1. Select **Evidence**. **Compliance over time** shows satisfied and unsatisfied results. **Agent health** shows agents sending heartbeats over time.
2. Type a label query and select **Search**. For example, the recordings use:

   ```text
   environment=showcase
   ```

3. Select a result to open its detail page. If your API supports subjects, use **Subject: any** to narrow the search to one subject.

![Search evidence by label, inspect the subject selector, export results, and open a record.](/img/ui/2.12/search-evidence.gif)

Queries compare labels with `=` or `!=`, joined with `AND` or `OR`. Group conditions with parentheses when you mix the operators:

```text
environment=production AND (_plugin=apt-versions OR _plugin=local-ssh)
```

Conditions without an operator between them use `AND`. Mixing `AND` and `OR` at the same level without parentheses is an error.

The query is part of the page's address. **Share** copies that address, and **Export to CSV** downloads results. Use the three-dot menu to show or hide labels, including hidden labels whose names start with `_`.

![Filter by a subject, share the search, and show hidden evidence labels.](/img/ui/2.12/evidence-search-actions.gif)

### Review the overview and tasks

Open a result and select **Overview**. The summary shows its status, collection time and expiration. **Current State** shows the status reason and metadata counts. **Subjects** identifies what the evidence is about. **View Tasks** opens the activities recorded with the check.

![Review the evidence status, subject and current state, then open its recorded tasks.](/img/ui/2.12/evidence-overview.gif)

### Inspect the policy evaluation

When the evidence includes policy artifacts, the API replays the evaluation. On **Overview**, expand **Violations**, **Policy**, **Config (policy data)**, **Input** and **Evaluation output** to inspect it. A violation can link to the policy rule that produced it. If the replay differs from the recorded result, the page reports the difference.

![Inspect a recorded violation and its policy rule, then expand the policy data, input and evaluation output.](/img/ui/2.12/evidence-policy-replay.gif)

### Read metadata

Select **Metadata** to inspect the record's labels, properties and links. Labels help you construct searches. Properties carry additional check details, and links point to related resources.

![Open Metadata and scroll through evidence labels, properties and links.](/img/ui/2.12/evidence-metadata.gif)

### Inspect associated risks

Select **Risks** to see **Associated Risks** for the stream across system security plans (SSPs). Each linked risk shows its SSP and status. Select **Create Risk** and choose an SSP to open the risk form for that system.

![Review a linked risk, open Create Risk, and choose the fictional portal SSP.](/img/ui/2.12/evidence-risks.gif)

### Read attached media

Select **Media** to inspect resources linked from the evidence's back matter. Expand a text attachment to read it. Use its download icon to save it.

![Expand the fictional port report on Media and download the text attachment.](/img/ui/2.12/evidence-media.gif)

### Verify the signature

Select **Signature**, then **Verify**. **Verification** compares the current evidence content with its stored signed payload. Review the individual check results and **Signature Details**, including the signer and content hash. **Re-verify** runs the checks again.

![Run signature verification and inspect its check results, content hash and signer details.](/img/ui/2.12/evidence-signature.gif)

### Follow the stream's history

Select **History** to see **Compliance over time**, **Agent health** and earlier observations. Select **View** beside an older record to inspect it. On that record, **Go to latest evidence** returns you to the newest observation.

![Open History, view an earlier evidence record, and return to the latest evidence.](/img/ui/2.12/evidence-history.gif)

### Re-submit evidence

Select **Re-submit** to open a form prefilled from the record. Submitting it creates a new revision in the same stream and makes that revision the latest evidence. Review the fields before submitting; the recording shows the form without submitting a new revision.

![Open Re-submit and review the prefilled evidence revision form.](/img/ui/2.12/resubmit-evidence.gif)

### Save an evidence filter

1. On **Evidence**, run a search and select **Save**.
2. In **New Evidence Filter**, enter a **Name**. **Result Filter** holds your query.
3. Choose **Scope**: **Global**, or one SSP.
4. Choose the **Controls** the filter supports. When `FILTER_COMPONENT_LINKING_ENABLED` is on, you can link **Components** instead; see the [configuration reference](../configuration.md).
5. Select **Submit** to save the filter and return to the evidence search.

![Save the showcase search as a named evidence filter linked to a control.](/img/ui/2.12/save-evidence-filter.gif)

### Manage evidence filters

Select **Evidence Filters** to list saved filters. Narrow the list by **Scope**, **Control** or **Component**. Use **Edit** to change a filter and **Delete** to remove it after confirming.

![Inspect the filter list, edit a saved filter, and open its deletion confirmation.](/img/ui/2.12/manage-evidence-filters.gif)

You can also link filters from the **Controls** page; see [Controls and the Compliance Map](./controls.md). If your API has AI configured, review suggested filters under **Evidence filter suggestions** and **Accept** or **Reject** them. Without AI, the page says **AI is not configured**.

## Troubleshooting

- If a search has no results, check the label names and values. Clear a subject selection to search across subjects.
- If **Media** has no items, the record has no displayable linked resources.
- If the policy evaluation is unavailable, the record may lack the artifacts needed for replay.
- If verification fails, inspect the reported checks and errors before relying on the record.

## Related

- [Controls and the Compliance Map](./controls.md)
- [Risks](./risks.md)
- [Agent overview](/agent/0.9/)
