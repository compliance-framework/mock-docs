---
title: Mock UI overview
description: Mock content for the CCF Factory's tests. The Mock UI 0.1 line documents no real software.
sidebar_position: 1
sidebar_label: Overview
---

# Mock UI overview

:::caution[Mock content]

This page is placeholder content in `compliance-framework/mock-docs`, the CCF Factory's safe target. It documents no real software, and its versions are fake. For the real documentation, see the [CCF docs](https://compliance-framework.github.io/docs/).

:::

The `mock-ui` repository is a stand-in for the CCF UI. It pretends to show assessments, controls and evidence. This page is for **developers** who test the CCF Factory against it. It documents the fake 0.1 line.

## How it works

1. The mock UI serves one static page on `MOCK_UI_PORT`.
2. It reads `MOCK_UI_API_URL` but never calls it.

## Use it

```sh
mock-ui serve
```

Set it up with the environment variables in the [configuration reference](./configuration.md).

## Related

- [Configuration reference](./configuration.md)
- [`compliance-framework/mock-ui`](https://github.com/compliance-framework/mock-ui)
