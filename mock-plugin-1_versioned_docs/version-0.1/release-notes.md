---
title: Release notes
description: Release notes for Mock Plugin 1 in the 0.1 line.
sidebar_position: 2
---

# Release notes

For operators and developers using `mock-plugin-1` in the Continuous Compliance Framework test environment.

## v0.1.4 (2026-10-14)

### Highlights

The served plugin now requires `Configure` before `Init`. A probe that calls only `Init` exits with status `1` as part of the gate failure test. ([#40](https://github.com/compliance-framework/mock-plugin-1/pull/40))

### Changed

- Adds a gate failure test: the served plugin logs an error and exits with status `1` if you call `Init` before `Configure`. ([#40](https://github.com/compliance-framework/mock-plugin-1/pull/40))

### Fixed

- Updates Go dependencies after the Renovate test pins. ([#38](https://github.com/compliance-framework/mock-plugin-1/pull/38))
