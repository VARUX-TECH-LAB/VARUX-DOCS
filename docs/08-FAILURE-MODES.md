---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---


# Failure Modes

## Purpose

This document defines how AXIS behaves under failure in Native PG mode.

## Rule

If AXIS is uncertain about safety, it must fail closed and preserve evidence where possible.

## Failure Matrix

| Failure | Behavior |
|---|---|
| Backend down before query | ErrorResponse, audit, no forward |
| Backend timeout before completion | EXECUTION_UNKNOWN if bytes sent |
| Audit WAL unavailable | Reject protected operations |
| Policy engine unavailable | BLOCK |
| Policy manifest invalid | Refuse readiness |
| Client disconnect before forward | audit disconnect, no backend action |
| Client disconnect after forward | backend may continue; audit state accordingly |
| Backend response lost | EXECUTION_UNKNOWN |
| Backend completed but client disconnected | CLIENT_DELIVERY_UNKNOWN |
| Partial write to backend | EXECUTION_UNKNOWN + close |
| Partial response from backend | close + audit |
| AXIS OOM/crash | covered by restart recovery semantics |
| Disk full | readiness fail + reject protected operations |
| Protocol unsupported | FeatureNotSupported or close |
| Transaction poisoned | safety rollback attempt + close |
| Backend rollback failure | TRANSACTION_RESET_FAILED + close |

## Recursive Audit Failure

If AXIS cannot write audit evidence, it cannot safely process protected writes.

Behavior:

1. Circuit breaker opens.
2. Readiness fails.
3. New protected operations are rejected.
4. Operator alert is emitted.
5. Recovery requires audit subsystem health restoration.

## Execution Unknown

Use when AXIS sent bytes to backend but cannot confirm whether backend completed.

Required fields:

```json
{
  "bytes_written_to_backend": 142,
  "backend_response_received": false,
  "cannot_confirm_non_execution": true
}
```

## Client Delivery Unknown

Use when backend completed but the client did not receive confirmation.

Required fields:

```json
{
  "backend_completed": true,
  "client_delivery_confirmed": false,
  "database_state_known": true
}
```

## Failure During Safety ROLLBACK

If AXIS attempts rollback and fails:

- Mark connection poisoned.
- Close client connection.
- Close backend connection if possible.
- Audit reset failure.
- Increment high-severity metric.
- Require operator visibility.

## Current Known Weaknesses

- AXIS crash cannot write events after death; recovery must reconcile incomplete evidence.
- Backend-side idle transaction cleanup must be configured.
- Some failures require manual verification.

## Acceptance Criteria

Every failure mode must produce either durable evidence or a documented absence-of-evidence state with operator guidance. Pretending the problem is solved because the process restarted is how humans keep inventing outages with nicer dashboards.
