---
title: Configuration reference
description: Mock content for the CCF Factory's tests. The environment variables of the fake Mock UI 0.0 line.
sidebar_position: 2
---

# Configuration reference

:::caution[Mock content]

This page is placeholder content in `compliance-framework/mock-docs`, the CCF Factory's safe target. It documents no real software, and its versions are fake. For the real documentation, see the [CCF docs](https://compliance-framework.github.io/docs/).

:::

This page is for **operators** of the mock. It lists the environment variables `mock-ui` reads when it starts, in the fake 0.0 line.

| Setting | Purpose | Type | Default | Example |
|---|---|---|---|---|
| `MOCK_UI_API_URL` | The mock API the mock UI would talk to. | `string` | *required* | `MOCK_UI_API_URL=http://localhost:8080` |
| `MOCK_UI_PORT` | The port the mock UI serves on. | `int` | `3000` | `MOCK_UI_PORT=3001` |
