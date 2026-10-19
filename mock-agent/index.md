---
title: Mock Agent overview
description: Mock content for the CCF Factory's tests. The Mock Agent 0.1 line documents no real software.
sidebar_position: 1
sidebar_label: Overview
---

# Mock Agent overview

:::caution[Mock content]

This page is placeholder content in `compliance-framework/mock-docs`, the CCF Factory's safe target. It documents no real software, and its versions are fake. For the real documentation, see the [CCF docs](https://compliance-framework.github.io/docs/).

:::

The `mock-agent` repository is a stand-in for the CCF Agent. It pretends to run plugins and send their evidence to the mock API. This page is for **developers** who test the CCF Factory against it. It documents the fake 0.1 line.

## How it works

1. The mock agent wakes up every `MOCK_AGENT_INTERVAL`.
2. It logs that it ran its plugins, and sends nothing to `MOCK_AGENT_API_URL`.

## Use it

```sh
mock-agent agent
```

Set it up with the environment variables in the [configuration reference](./configuration.md).

## Related

- [Configuration reference](./configuration.md)
- [`compliance-framework/mock-agent`](https://github.com/compliance-framework/mock-agent)
