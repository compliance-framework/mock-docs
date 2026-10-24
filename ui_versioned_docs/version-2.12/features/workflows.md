---
title: Workflows
description: Define recurring compliance activities in the UI, run them for a system and complete the tasks they assign you.
sidebar_position: 5
---

# Workflows

Use workflows in the Continuous Compliance Framework (CCF) to schedule compliance activities and complete the tasks they assign you. This guide is for compliance and GRC users.

## How it works

- A **workflow definition** is the template: its steps, the role responsible for each step, the evidence each step requires, and a suggested cadence.
- A **workflow instance** runs a definition for one system security plan (SSP), on its own cadence, with users assigned to the definition's roles.
- Each run of an instance is an **execution**. Its steps become tasks for the users who hold the step's role, and show up under **My Tasks**.

## Before you start

- The recordings use UI `v2.12.1` and API `v0.21.1` in a temporary environment with fictional data.
- To see definitions and instances, you need `workflow-definition:read` and `workflow-instance:read`.
- To follow the creation flows, you also need `create` permission for `workflow-definition`, `workflow-step-definition`, `workflow-instance`, `role-assignment` and `workflow-execution`, plus read access to their details and users. The `contributor` role includes these permissions.
- Select an active system before creating an instance. See [Systems](./systems.md).
- For scheduled runs, the API's workflow scheduler must be on: see `CCF_WORKFLOW_CONFIG` in the [API configuration reference](/api/configuration).

## Use it

### Create a workflow definition

Select **Workflows** > **Workflow Definitions**, then **New Definition**. Enter a **Name**, **Description**, **Version**, **Suggested Cadence** and **Grace Period (Days)**. Choose **Required Evidence Types** if needed, then select **Create Definition**.

![Create a workflow definition, enter its name and description, choose a quarterly cadence, and save it.](/img/ui/2.12/create-workflow-definition.gif)

The definition's **Overview** tab lets you update these settings. On **Controls**, select **Link Control** to link the controls the workflow supports.

### Add a workflow step

Open the definition's **Steps** tab and select **Add Step**. Enter a **Step Name**, **Description** and **Responsible Role**, then select **Create Step**. The recording uses the role `Access reviewer`.

![Open Steps, add a review step, set its responsible role and grace period, and save the step.](/img/ui/2.12/create-workflow-step.gif)

Use **Evidence Required** and **Evidence Requirements** to specify the evidence a step needs. When other steps exist, **Dependencies** lets you choose which must complete first. A dependent step stays blocked until those steps are complete.

### Create a workflow instance

Select **Workflows** > **Workflow Instances**, then **New Instance**. Choose the **Workflow Definition**, enter an **Instance Name**, and choose its **Execution Cadence** and **Grace Period (Days)**. Select **Create Instance** to save it for the active system.

![Create an instance for the fictional Lantern Portal, select the workflow definition, enter a name, and choose a quarterly cadence.](/img/ui/2.12/create-workflow-instance.gif)

For a custom cadence, enter a six-field **Cron Expression (6 fields)**: second, minute, hour, day, month, weekday.

### Assign roles to the instance

Open the instance's **Role Assignments** tab and select **Assign Role**. Search for a user for each role the definition's steps use. Select a search result, then **Assign Selected Roles** to save the assignments.

![Open Role Assignments, search for the fictional user Avery Example, select the user, and assign the Access reviewer role.](/img/ui/2.12/assign-workflow-roles.gif)

Assign roles before starting a run so its steps have task owners.

### Execute a workflow

On the instance, select **Execute Now**, then confirm with **Start Execution**. The execution page shows step states and progress. Return to the instance's **Executions** tab to find the run and select **View** to inspect it.

![Start a workflow execution, confirm the run, inspect its progress, and find it in the instance execution history.](/img/ui/2.12/execute-workflow.gif)

An active instance also runs on its cadence. **Deactivate** stops scheduled runs, and **Activate** resumes them. The execution page lets you **Cancel Execution** or **Retry Execution** when those actions apply.

### Complete your tasks

1. Open **My Tasks** from the profile menu. It lists the steps assigned to you, with their due dates. Narrow them by **Status** and due date.
2. Open a task and select **Start Step**.
3. Add the evidence the step requires with **Add Evidence**. Evidence is submitted when you complete the step.
4. Select **Complete Step**, with optional **Completion Notes**.

If you can't complete a step, select **Mark Failed** and give a reason; this can block the steps that depend on it. To hand a task to someone else, select **Reassign Task**.

## Troubleshooting

- If **Workflow Instances** says **System Security Plan not selected**, select a system first.
- If a role has no task owner, check the instance's **Role Assignments** before starting the next execution.
- If a step is blocked, check whether its dependencies are complete.

## Related

- [Systems](./systems.md)
- [Evidence](./evidence.md)
- [UI overview](../index.md)
