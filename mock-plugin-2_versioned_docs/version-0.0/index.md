---
title: Mock Plugin 2
description: Mock content for the CCF Factory's tests. The Mock Plugin 2 0.0 line documents no real plugin.
sidebar_position: 1
---

# Mock Plugin 2

:::caution[Mock content]

This page is placeholder content in `compliance-framework/mock-docs`, the CCF Factory's safe target. It documents no real software, and its versions are fake. For the real documentation, see the [CCF docs](https://compliance-framework.github.io/docs/).

:::

`mock-plugin-2` pretends to collect evidence about a set of user accounts. This page is for **operators** who run it with the mock agent, and for **developers** who write mock policies for it. It documents the fake 0.0 line.

## Requirements

None. The plugin reads no real system and needs no credentials.

## Configuration

| Setting | Purpose | Type | Default | Example |
|---|---|---|---|---|
| `directory` | The pretend user directory to read. | `string` | *required* | `directory: mock-directory` |

## Evidence collected

One pretend evidence record per item, each time the agent runs the plugin.

## Policies

The policy bundles that work with this plugin, at their newest releases.

| Bundle | Latest version | Controls covered | Compatible plugin lines |
|---|---|---|---|
| [`mock-plugin-policies-2`](https://github.com/compliance-framework/mock-plugin-policies-2) | `v0.1.0` | MOCK-3 | latest |
