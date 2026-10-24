---
title: Control definitions
description: Manage control catalogs, profiles and the links between controls in the UI.
sidebar_position: 6
---

# Control definitions

Use control definitions in the Continuous Compliance Framework (CCF) to manage catalogs of requirements, profiles that select controls, and links between controls. This page is for **compliance and GRC users** who maintain them.

## How it works

- A **catalog** holds controls, in groups. The UI sorts catalogs into **Standards**, **Policies**, **Procedures**, **Internal Catalogs** and **Other**.
- A **profile** imports controls from one or more catalogs. A system security plan (SSP) attaches profiles to say which controls it must meet.
- A **control link** relates a control to another: a policy control **Implements** a standard's control, or a procedure **Documents** a policy. The **Compliance Map** follows these links; see [Controls and the Compliance Map](./controls.md).

## Before you start

- The recordings use UI `v2.12.1` and API `v0.21.1` in a temporary environment with fictional data.
- To see them, you need `catalog:read`, `profile:read` and `control-link:read`.
- To change them, a role that can change OSCAL documents, such as `contributor`.
- To bring in published catalogs and profiles, such as NIST SP 800-53, import their OSCAL JSON under **Admin** > **Import**. See [Administration](./administration.md).

## Use it

### Create a catalog

Select **Control Definitions** > **Catalogs**. Each tab lists one kind of catalog, with **View**, **JSON** and **Delete**. **Create Catalog** starts a new one.

Enter a **Title**, generate an **ID**, and choose the **Type**. Select **Create Catalog** to save it. The recording creates a **Standard**.

![Create a fictional standard catalog, generate its UUID, enter its title, and save it.](/img/ui/2.12/create-catalog.gif)

### Create a group

In the catalog, select **Add Group**. Enter the group's **ID** and **Title**, then select **Create Group**. Expand the saved group to see its contents.

![Create an access management group inside the fictional Orchard catalog.](/img/ui/2.12/create-control-group.gif)

### Create a control

Select **Control** beside a group to create a control inside it. Enter an **ID** and **Title**, then select **Create Control**. To create a control at the catalog's top level, use **Add Control** instead.

![Expand a group, enter a control ID and title, and create the control inside the group.](/img/ui/2.12/create-control.gif)

### Add a statement

Expand the control and select **Add Statement**. Enter the requirement in **Text**, then select **Save Text**. Reopen the catalog to verify the saved statement.

![Expand a control, add its requirement text, save it, and reopen the catalog to read the statement.](/img/ui/2.12/add-control-statement.gif)

### Create a profile

Select **Control Definitions** > **Profiles**, then **Create New Profile**. Enter a **Title** and optional **Remarks**, and select **Create Profile**.

![Create a fictional Orchard baseline profile with a title and remarks.](/img/ui/2.12/create-profile.gif)

### Add controls to a profile

On the profile's **Controls** tab:

1. Select **Add Catalog Import**, then **Import** beside the catalog you want.
2. Expand the imported catalog. Under **Included Controls**, select **Create Group**, then expand **Group 1**.
3. Search for a control by ID, select the matching suggestion, and select **Add**. Repeat for each control you want to include.
4. Select **Save**. Reopen the profile to verify the **Included Controls** count.

![Import the Orchard catalog, select a control under Included Controls, save, and verify the profile.](/img/ui/2.12/add-profile-controls.gif)

Use **Excluded Controls** to omit selected controls from an import. **Build Profile by Props** builds a profile from controls whose props match your rules, for example every control with a given `class`. A rule compares a prop with `equals`, `contains`, `regex` or `in`, and **Match Strategy** says whether `any` or `all` rules must match.

A profile's **Compliance** tab shows the compliance of its controls, and its **JSON** tab its OSCAL document.

### Link two controls

Select **Control Definitions** > **Control Links**, then **Create Control Link**:

1. Set **Link scope** to **Single control**.
2. Choose the source: the concrete side, which implements or documents the target.
3. Choose the **Relationship**. The form shows the direction it allows.
4. Choose the **Target control**: the more abstract control.
5. Select **Create Control Link** and verify the saved relationship in the list.

The recording links a fictional policy control to the standard control it **Implements**.

![Choose a policy control, select Implements, choose the target standard control, and save the link.](/img/ui/2.12/link-controls.gif)

To link every control of a source catalog, set **Link scope** to **Whole catalog**. **Re-sync** updates it when the source catalog's controls change.

## Related

- [Systems](./systems.md)
- [Controls and the Compliance Map](./controls.md)
- [OSCAL](/oscal)
