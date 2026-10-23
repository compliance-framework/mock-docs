---
title: API release notes
description: Patch releases for the Continuous Compliance Framework API 0.21 line.
---

# API release notes

These notes are for operators and developers who run or integrate with the Continuous Compliance Framework API `0.21` line.

## v0.21.1 (2026-10-14)

### Highlights

- When several evidence streams resolve at the same time, the risk evidence worker locks the risk before removing links and checking its status. This prevents concurrent resolutions from leaving a risk open after its last evidence link is removed. ([#490](https://github.com/compliance-framework/api/pull/490))

### Fixed

- Read the risk status under a database lock before resolving an evidence link. Skip risks that have already closed or reached `remediated`. ([#490](https://github.com/compliance-framework/api/pull/490))
- Update `github.com/labstack/echo/v4` to `v4.15.3` for a security dependency fix. ([#491](https://github.com/compliance-framework/api/pull/491))
- Update `github.com/moby/go-archive` to `v0.3.0` for a security dependency fix. ([#492](https://github.com/compliance-framework/api/pull/492))
- Update the Go cryptography, module, and networking dependencies for security fixes. ([#494](https://github.com/compliance-framework/api/pull/494), [#495](https://github.com/compliance-framework/api/pull/495), [#501](https://github.com/compliance-framework/api/pull/501))
