---
title: Configuration reference
description: Mock content for the CCF Factory's tests. The environment variables of the fake Mock API 0.1 line.
sidebar_position: 2
---

# Configuration reference

:::caution[Mock content]

This page is placeholder content in `compliance-framework/mock-docs`, the CCF Factory's safe target. It documents no real software, and its versions are fake. For the real documentation, see the [CCF docs](https://compliance-framework.github.io/docs/).

:::

This page is for **operators** of the mock. It lists the environment variables `mock-api` reads when it starts, in the fake 0.1 line.

| Setting | Purpose | Type | Default | Example |
|---|---|---|---|---|
| `MOCK_API_LOG_FORMAT` | The log format: `text` or `json`. *(new in 0.1)* | `string` | `text` | `MOCK_API_LOG_FORMAT=json` |
| `MOCK_API_LOG_LEVEL` | How much the mock API logs: `debug`, `info`, `warn` or `error`. | `string` | `info` | `MOCK_API_LOG_LEVEL=debug` |
| `MOCK_API_PORT` | The port the mock API listens on. | `int` | `8080` | `MOCK_API_PORT=9090` |
