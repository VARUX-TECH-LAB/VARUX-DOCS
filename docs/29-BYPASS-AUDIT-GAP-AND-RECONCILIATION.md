---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# Bypass Audit Gap and Reconciliation

## Purpose

This document defines how AXIS records, limits, and reconciles audit gaps when emergency bypass or degraded modes are used.

Bypass is operationally necessary, but it weakens AXIS evidence guarantees. This must be declared, not hidden under soothing language.

## Audit Gap Types

| Gap Type | Meaning |
|---|---|
| hard_bypass_gap | AXIS not inline; sees no traffic |
| audit_only_gap | AXIS inline but not enforcing |
| partial_visibility_gap | AXIS sees only some services |
| crash_gap | AXIS process died before completion audit |
| export_gap | evidence exists but was not exported before bypass |

## Hard Bypass Evidence Status

During hard bypass, AXIS cannot prove:

- queries were evaluated;
- dangerous writes were blocked;
- backend non-execution for blocked operations;
- approval requirements were enforced.

Evidence must shift to customer-provided sources:

- PostgreSQL native logs;
- pg_audit;
- application logs;
- connection pool logs;
- change management records;
- operator incident notes.

## Audit-Only Mode

Audit-only mode is a possible future degraded mode.

Requirements:

- AXIS remains inline.
- AXIS records queries and decisions.
- AXIS does not enforce selected policies.
- Every event includes `enforcement_mode="audit_only"`.
- UI and health endpoints clearly show degraded mode.
- Mode requires operator authorization and expiry.
- Audit-only mode cannot be silently enabled by config drift.

Audit-only mode should be used only when business continuity outweighs enforcement.

## Reconciliation Procedure

After bypass:

1. Record bypass start/end times.
2. Export AXIS pre-bypass bundle.
3. Collect PostgreSQL logs for window.
4. Collect application/pool logs.
5. Identify high-risk operations.
6. Compare operations against AXIS policy as offline analysis where possible.
7. Mark unverifiable operations.
8. Produce reconciliation report.
9. Append `AXIS_BYPASS_RECONCILIATION_COMPLETED` to audit WAL.
10. Review incident and update runbook.

## Reconciliation Report Schema

```json
{
  "schema_version": "axis.bypass.reconciliation.v1",
  "bypass_id": "bypass_...",
  "window_start": "...",
  "window_end": "...",
  "mode": "hard_bypass",
  "axis_audit_head_before": "...",
  "axis_audit_head_after": "...",
  "postgres_log_sources": ["..."],
  "operations_seen": 1234,
  "high_risk_operations": 4,
  "unverified_operations": 2,
  "manual_findings": [],
  "reviewed_by": "operator_id"
}
```

## Customer Contract Language

Customer-facing documents must state:

- hard bypass disables AXIS enforcement;
- hard bypass creates an AXIS evidence gap;
- AXIS can help reconcile but cannot prove what it did not observe;
- PostgreSQL logging must be enabled during bypass;
- emergency bypass must be time-bounded.

## Current Known Weaknesses

- Reconciliation cannot recreate AXIS non-execution proof.
- PostgreSQL logs may be insufficient without prior configuration.
- Audit-only mode is not implemented in PoC.
- Offline policy replay may not perfectly reconstruct runtime context.

## Success Looks Like

After bypass, everyone knows exactly what evidence was lost, what was reconstructed, and what remains unverified.

## Failure Looks Like

A bypass window disappears from the story because acknowledging it is uncomfortable. Discomfort is not a logging strategy.
