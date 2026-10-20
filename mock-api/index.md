---
title: Mock API overview
description: Mock content for the CCF Factory's tests. The Mock API 0.1 line documents no real software.
sidebar_position: 1
sidebar_label: Overview
---

# Mock API overview

:::caution[Mock content]

This page is placeholder content in `compliance-framework/mock-docs`, the CCF Factory's safe target. It documents no real software, and its versions are fake. For the real documentation, see the [CCF docs](https://compliance-framework.github.io/docs/).

:::

The `mock-api` repository is a stand-in for the CCF API. It pretends to store compliance data and take in evidence from agents. This page is for **developers** who test the CCF Factory against it. It documents the fake 0.1 line.

## How it works

1. The mock API starts one HTTP server on `MOCK_API_PORT`.
2. It answers every request with a fixed, empty response.
3. It keeps nothing: restarting it loses no data, because it stores none.

## Use it

```sh
mock-api run
```

Set it up with the environment variables in the [configuration reference](./configuration.md).

## Related

- [Configuration reference](./configuration.md)
- [`compliance-framework/mock-api`](https://github.com/compliance-framework/mock-api)
