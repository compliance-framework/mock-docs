---
title: Navigation
description: The entries of the UI's sidebar, what each one opens and the permission it needs.
sidebar_position: 3
---

# Navigation

This page is for **compliance and GRC users** and **operators**. It lists the released sidebar entries in UI `v2.12.1`, followed by the header, profile menu and page tabs. The Continuous Compliance Framework (CCF) uses the API to authorize the data behind each page.

The sidebar hides an entry you don't have the permission for, and a group whose entries are all hidden. The **Evidence Filters** list and guarded **Admin** routes also check permissions when you open their address directly, and send you back to the start page, **Evidence**. Permissions are written `resource:action`, the way the API's `/api/me/permissions` endpoint names them. For the roles that grant them, see [Authentication and permissions](/api/authentication-and-permissions).

To collapse the sidebar to abbreviations, select **Toggle Sidebar** on its edge. Hover over an abbreviation to see the entry's name.

| Entry | Path | What it opens | Sidebar permission |
|---|---|---|---|
| **Systems** > **Active System** | `/system` | The active system security plan (SSP): its overview, components, risks, compliance and JSON. See [Systems](./features/systems.md). | No sidebar gate |
| **Systems** > **Manage Systems** | `/system-security-plans` | The list of SSPs, where you create one and choose the active one. | `ssp:read` |
| **Evidence** | `/evidence` | Search the evidence agents and other sources sent, by label. See [Evidence](./features/evidence.md). | `evidence:read` |
| **Controls** | `/controls` | How the active SSP implements each control of its profile. See [Controls and the Compliance Map](./features/controls.md). | No sidebar gate |
| **Compliance Map** | `/lineage` | The lineage from standards to policies, controls and evidence, with compliance and open risk at each step. | No sidebar gate |
| **Risk Register** | `/risks` | The risks of the active SSP. See [Risks](./features/risks.md). | `risk:read` |
| **Evidence Filters** | `/dashboards` | Saved evidence searches, and the controls and components they're linked to. See [Evidence](./features/evidence.md#evidence-filters). | `filter:read` |
| **Workflows** > **Workflow Definitions** | `/workflows` | The templates of recurring compliance activities. See [Workflows](./features/workflows.md). | `workflow-definition:read` |
| **Workflows** > **Workflow Instances** | `/workflow-instances` | The workflows that run for the active SSP. | `workflow-instance:read` |
| **Control Definitions** > **Catalogs** | `/catalogs` | Control catalogs. See [Control definitions](./features/control-definitions.md). | `catalog:read` |
| **Control Definitions** > **Control Links** | `/control-links` | Relationships between controls, such as one control implementing another. | `control-link:read` |
| **Control Definitions** > **Profiles** | `/profiles` | Profiles: baselines of controls selected from catalogs. | `profile:read` |
| **Admin** > **Risks** | `/admin/risks` | The risks of every SSP. See [Administration](./features/administration.md). | `admin:manage` |
| **Admin** > **System Users** | `/users` | Users who sign in with a password or single sign-on. | `admin:manage` |
| **Admin** > **Groups** | `/admin/groups` | Groups of users. | `admin:manage` |
| **Admin** > **Agents** | `/admin/agents` | Registered agents, their service account keys and their remote configuration. | `agent:read` |
| **Admin** > **Diagnostics** | `/admin/diagnostics` | Notification and AI suggestion diagnostics. | `admin:manage` |
| **Admin** > **Import** | `/admin/import` | Import OSCAL documents and evidence filters from JSON files. | `admin:manage` |
| **Logout** | `/auth/logout` | Signs you out. It's at the bottom of the sidebar. | No sidebar gate |

The permission column controls sidebar visibility, not permission to perform every action on the page. Entries without a sidebar gate still require sign-in and API access. **Agents** checks `agent:read` on its route too. A user detail link can open outside the admin user list, subject to API authorization.

## Header and personal pages

| Entry | Where to find it | What it does |
|---|---|---|
| **Active System** | Header, after you select a system | Shows the selected SSP's name. To change it, open **Systems** > **Manage Systems** and select **Set Active**. **View** alone does not select it. |
| **My Tasks** | Profile menu at the top right | Opens `/my-tasks` for your workflow assignments. The badge shows available tasks. See [Workflows](./features/workflows.md#complete-your-tasks). |
| **Preferences** | Profile menu | Opens `/preferences` for your notification channels and Slack account link. See [Preferences](./index.md#preferences). |
| **Logout** | Profile menu or sidebar footer | Ends your UI session. |

The profile menu uses your initials as its button. These personal pages require sign-in; their menu entries have no admin gate. The API still controls which assignments and account data you can access.

## Pages within a section

A detail page's tabs are separate from the sidebar. Open the parent entry first, then a record. For system-scoped pages, select an SSP first; **Active System** displays **System Security Plan not selected** until you do.

| Entry path | Visible sections | Guide |
|---|---|---|
| **Systems** > **Active System**, or **Manage Systems** > **View** | **Overview**, **Components**, **Risks**, **Compliance**, **JSON** | [Systems](./features/systems.md) |
| **Control Definitions** > **Profiles**, then a profile | **Controls**, **Compliance**, **JSON** | [Control definitions](./features/control-definitions.md) |
| **Workflows** > **Workflow Definitions**, then a definition | **Overview**, **Steps**, **Controls**, **JSON** | [Workflows](./features/workflows.md) |
| **Workflows** > **Workflow Instances**, then an instance | **Overview**, **Role Assignments**, **Executions**, **JSON** | [Workflows](./features/workflows.md) |
| **Admin** > **Diagnostics** | **Notifications**; **AI Suggestions** when AI is enabled | [Administration](./features/administration.md#diagnostics) |
| **Admin** > **Agents**, then an agent | **Details**, **Service Account Keys** for admins, **Configuration** when allowed | [Administration](./features/administration.md#agents) |

### Routes without released menu entries

In `v2.12.1`, **Inventory**, **Assessment Plans**, **Assessment Results** and **POA&M** are commented out of the sidebar. So are **Component Definitions**, **Parties**, **Roles**, **Subject Templates** and **Risk Templates** under **Admin**. Their routes exist, but you cannot reach them through those sidebar entries. Do not use route existence as evidence that a section is available from the menu.

The SSP tab bar does not expose **System Characteristics**, **System Implementation**, **Users**, **Leveraged Authorizations**, **Control Implementation**, **Export Offerings**, **Leverage** or **Inherited Capabilities**. Use the sidebar's **Controls** entry for the released control implementation workspace. Profiles also omit the **Merge** tab. These omissions are part of this release's navigation, not permissions you can grant to reveal the tabs.

## Related

- [UI overview](./index.md)
- [Authentication and permissions](/api/authentication-and-permissions)
