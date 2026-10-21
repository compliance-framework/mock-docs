---
title: Release notes
description: Mock Plugin 2 releases in the 0.1 line.
sidebar_position: 2
---

# Release notes

These notes are for operators and developers using `mock-plugin-2` to test Continuous Compliance Framework release automation.

## v0.1.4 (2026-10-14)

### Highlights

This mock release includes the grouped Go dependency security update following the dependency-pinning test. ([#39](https://github.com/compliance-framework/mock-plugin-2/pull/39))

### Fixed

- The grouped update sets `golang.org/x/crypto` to `v0.57.0`, `golang.org/x/net` to `v0.60.0`, `golang.org/x/text` to `v0.42.0`, and `google.golang.org/grpc` to `v1.83.2`. ([#39](https://github.com/compliance-framework/mock-plugin-2/pull/39))
