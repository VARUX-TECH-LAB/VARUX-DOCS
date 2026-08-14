---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---


# Restart Recovery Semantics

## Purpose

This document defines how AXIS handles its own crash, restart, OOM kill, node eviction, or rolling upgrade.

## Problem

If AXIS dies mid-flight, it may be unable to write final audit events. Backend queries may continue, transactions may remain open, and clients may time out.

A graceful drain plan does not solve OOM kill.

## Required Session Markers

Before forwarding a protected operation, AXIS should write an in-flight marker:

```json
{
  "event": "AXIS_BACKEND_DISPATCH_INTENT",
  "axis_request_id": "...",
  "session_id": "...",
  "backend_addr": "...",
  "sql_hash": "...",
  "durably_written_before_forward": true
}
```

After completion:

```json
{
  "event": "AXIS_BACKEND_COMPLETION_OBSERVED",
  "axis_request_id": "..."
}
```

On startup, AXIS scans for dispatch intents without completion.

## Startup Recovery

At startup:

1. Validate audit WAL continuity.
2. Identify incomplete in-flight operations.
3. Mark them as `RECOVERY_INCOMPLETE_EXECUTION`.
4. Emit operator alert.
5. Refuse readiness if unresolved critical uncertainty exceeds policy threshold.
6. Rely on backend-side idle transaction timeout as mitigation for leaked sessions.

## Backend Connection Leaks

AXIS process death should close its TCP sockets, but backend may keep work alive briefly. PostgreSQL timeout settings must be part of deployment:

- statement_timeout.
- idle_in_transaction_session_timeout.
- lock_timeout.
- TCP keepalive settings.

## Rolling Upgrade

Graceful path:

1. Mark instance draining.
2. Stop accepting new connections.
3. Allow safe idle connections to close.
4. Force-close poisoned/long-running sessions after timeout.
5. Write drain audit event.
6. Upgrade.
7. Rejoin after readiness.

## Current Known Weaknesses

- AXIS cannot write after sudden death.
- Recovery may only identify uncertainty, not resolve it.
- Backend-side timeout configuration is required.
- Global recovery in multi-AXIS mode requires instance IDs.

## Acceptance Criteria

AXIS restart semantics are acceptable only if an operator can see which operations were in-flight and which require manual verification.
