---
title: Agent release notes
description: Patch releases for the Continuous Compliance Framework Agent 0.9 line.
---

# Agent release notes

These notes are for operators who run the Continuous Compliance Framework (CCF) Agent `0.9` line.

## v0.9.2 (2026-10-14)

### Highlights

- CCF Agent updates its API dependency to `v0.21.1` and its `gooci` dependency to `v0.0.9`. ([#130](https://github.com/compliance-framework/agent/pull/130))

### Changed

- Use the API `v0.21.1` and `gooci` `v0.0.9` libraries. ([#130](https://github.com/compliance-framework/agent/pull/130))

### Fixed

- Update the `github.com/moby/go-archive` dependency to `v0.3.0` for a security fix. ([#119](https://github.com/compliance-framework/agent/pull/119))
- Update Go cryptography and networking dependencies for security fixes. ([#120](https://github.com/compliance-framework/agent/pull/120), [#121](https://github.com/compliance-framework/agent/pull/121))
- Update gRPC dependencies for security fixes; this release uses `google.golang.org/grpc` `v1.83.2`. ([#123](https://github.com/compliance-framework/agent/pull/123), [#129](https://github.com/compliance-framework/agent/pull/129))
