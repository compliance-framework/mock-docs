---
title: Release notes
description: Mock Plugin 1 releases in the 0.1 line.
sidebar_position: 2
---

# Release notes

These notes are for operators and developers using `mock-plugin-1` to test Continuous Compliance Framework release automation.

## v0.1.4 (2026-10-14)

### Highlights

This mock release deliberately makes initialization fail when you call `Init` before `Configure`, so you can test release-gate failure handling. ([#40](https://github.com/compliance-framework/mock-plugin-1/pull/40))

### Changed

- The served plugin exits with status `1` if `Init` runs before `Configure`. A probe that calls only `Init` therefore fails. Call `Configure` first when you initialize this version. ([#40](https://github.com/compliance-framework/mock-plugin-1/pull/40))

### Fixed

- The grouped Go dependency security update restores `golang.org/x/crypto` to `v0.57.0`, `golang.org/x/net` to `v0.60.0`, `golang.org/x/text` to `v0.42.0`, and `google.golang.org/grpc` to `v1.83.2` after the dependency-pinning test. ([#38](https://github.com/compliance-framework/mock-plugin-1/pull/38))
