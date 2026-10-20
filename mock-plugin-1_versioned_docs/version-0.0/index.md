---
title: Mock Plugin 1
description: Mock content for the CCF Factory's tests. The Mock Plugin 1 0.0 line documents no real plugin.
sidebar_position: 1
---

# Mock Plugin 1

:::caution[Mock content]

This page is placeholder content in `compliance-framework/mock-docs`, the CCF Factory's safe target. It documents no real software, and its versions are fake. For the real documentation, see the [CCF docs](https://compliance-framework.github.io/docs/).

:::

`mock-plugin-1` pretends to collect evidence about an inventory of servers. This page is for **operators** who run it with the mock agent, and for **developers** who write mock policies for it. It documents the fake 0.0 line.

## Requirements

None. The plugin reads no real system and needs no credentials.

## Configuration

| Setting | Purpose | Type | Default | Example |
|---|---|---|---|---|
| `inventory` | The pretend inventory to read. | `string` | *required* | `inventory: mock-inventory` |

## Evidence collected

One pretend evidence record per item, each time the agent runs the plugin.

## Policies

The policy bundles that work with this plugin, at their newest releases.

| Bundle | Latest version | Controls covered | Compatible plugin lines |
|---|---|---|---|
| [`mock-plugin-policies-1`](https://github.com/compliance-framework/mock-plugin-policies-1) | `v0.1.1` | MOCK-1, MOCK-2 | latest |
