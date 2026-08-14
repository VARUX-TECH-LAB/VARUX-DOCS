---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---


# RFC Lifecycle and Versioning

## Purpose

This document defines how the native PG RFC set evolves.

## Problem

RFCs rot. Implementation changes. Drivers change. PostgreSQL versions change. Six months later, everyone quotes a stale document in a meeting and calls it architecture. Charming, but no.

## Versioning

RFC versions:

- v1.0: initial design.
- v1.2: production-realism hardening.
- v1.2: post-PoC update.
- v2.0: Extended Query design accepted.
- v3.0: production pilot readiness.

## Required Metadata

Each document must include:

- status.
- applies_to.
- last_reviewed.
- owner.
- implementation_status.
- supersedes.
- related_tests.

## Change Control

RFC changes required when:

- protocol support changes.
- policy behavior changes.
- transaction behavior changes.
- audit event taxonomy changes.
- deployment topology changes.
- approval semantics change.
- performance claims change.
- customer pilot exposes mismatch.

## Review Gates

- Before PoC implementation.
- After PoC completion.
- Before Extended Query implementation.
- Before external reviewer package.
- Before customer pilot.
- After first incident or near miss.

## Current Known Weaknesses

- This v1.2 set still has draft status.
- Implementation may reveal missing protocol details.
- Responsibility owner must be assigned before team growth.

## Acceptance Criteria

No implementation merge may contradict RFC without updating RFC or explicitly recording a deviation.
