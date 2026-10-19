---
title: Configuration reference
description: Mock content for the CCF Factory's tests. The environment variables of the fake Mock Agent 0.1 line.
sidebar_position: 2
---

# Configuration reference

:::caution[Mock content]

This page is placeholder content in `compliance-framework/mock-docs`, the CCF Factory's safe target. It documents no real software, and its versions are fake. For the real documentation, see the [CCF docs](https://compliance-framework.github.io/docs/).

:::

This page is for **operators** of the mock. It lists the environment variables `mock-agent` reads when it starts, in the fake 0.1 line.

| Setting | Purpose | Type | Default | Example |
|---|---|---|---|---|
| `MOCK_AGENT_API_URL` | The mock API the mock agent would report to. | `string` | *required* | `MOCK_AGENT_API_URL=http://localhost:8080` |
| `MOCK_AGENT_INTERVAL` | How often the mock agent pretends to run its plugins. | `duration` | `5m` | `MOCK_AGENT_INTERVAL=1m` |
| `MOCK_AGENT_PLUGINS_DIR` | The directory the mock agent would load plugins from. *(new in 0.1)* | `string` | `./plugins` | `MOCK_AGENT_PLUGINS_DIR=/opt/mock/plugins` |
