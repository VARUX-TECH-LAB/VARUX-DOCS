---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# CancelRequest Design

## Purpose

This document defines how AXIS must handle PostgreSQL CancelRequest before production pilot.

CancelRequest is not a normal query message. It arrives on a separate connection and uses BackendKeyData `(pid, secret_key)` issued during startup. Mishandling it causes either broken cancellation or a security bypass.

## Why It Matters

Without CancelRequest support:

- applications cannot cancel long-running queries correctly;
- real backend cancellation keys may leak to clients;
- clients may bypass AXIS to cancel backend sessions;
- attackers may use leaked keys to disrupt other sessions;
- operators lose control during incidents.

## PoC Status

Simple Query PoC may reject or log CancelRequest as unsupported.

Production pilot must not begin until one of these is true:

1. CancelRequest mapping is implemented and tested; or
2. customer explicitly accepts that cancel is unsupported and config prevents backend key leakage.

Option 2 is unlikely to survive a serious reviewer, but optimism has injured systems before.

## Required Design

AXIS must not pass real backend `BackendKeyData` directly to the client in production mode.

Instead:

1. Backend sends real `(backend_pid, backend_secret)` to AXIS.
2. AXIS stores mapping internally.
3. AXIS generates fake `(axis_pid, axis_secret)`.
4. AXIS sends fake BackendKeyData to client.
5. Client later opens CancelRequest with fake key.
6. AXIS validates fake key.
7. AXIS looks up real backend key.
8. AXIS opens backend CancelRequest connection using real key.
9. AXIS audits cancel request and result.

## Mapping Table

```text
axis_pid, axis_secret
  -> session_id
  -> backend_addr
  -> backend_pid, backend_secret
  -> created_at
  -> expires_at
  -> state
```

State values:

- active;
- cancel_in_progress;
- cancelled;
- expired;
- backend_closed;
- unknown_after_restart.

## Security Requirements

- Fake secrets must be cryptographically random.
- Mapping must be per session.
- Mapping must expire when session closes.
- Mapping must not be reused.
- CancelRequest must be rate-limited.
- Failed cancel attempts must be audited.
- Real backend keys must never appear in client logs or ErrorResponse.

## Restart Behavior

Cancel mappings are process memory state unless persisted.

v1.2 recommendation:

- mappings are memory-only for early implementation;
- AXIS restart invalidates fake keys;
- clients attempting cancel after AXIS restart receive failure;
- backend-side timeout settings mitigate abandoned long queries;
- future production may persist encrypted mappings with strict expiry.

## Audit Events

Required events:

- `AXIS_BACKEND_KEYDATA_INTERCEPTED`
- `AXIS_BACKEND_KEYDATA_SUBSTITUTED`
- `AXIS_CANCEL_REQUEST_RECEIVED`
- `AXIS_CANCEL_REQUEST_VALIDATED`
- `AXIS_CANCEL_FORWARDED_TO_BACKEND`
- `AXIS_CANCEL_BACKEND_RESULT`
- `AXIS_CANCEL_REJECTED`

Example:

```json
{
  "event_type": "AXIS_CANCEL_REQUEST_RECEIVED",
  "event_body": {
    "axis_pid": 12345,
    "session_id": "sess_...",
    "validated": true,
    "forwarded_to_backend": true
  }
}
```

## Failure Modes

| Failure | Behavior |
|---|---|
| Unknown fake key | Reject, audit |
| Expired key | Reject, audit |
| Backend session closed | Reject or no-op, audit |
| Backend cancel connection fails | Audit cancel_unknown |
| AXIS restart lost mapping | Reject and suggest backend timeout/operator action |
| Cancel flood | Rate limit |

## Tests

- client receives fake BackendKeyData;
- real backend key never reaches client;
- valid fake cancel forwards real cancel;
- invalid fake cancel rejected;
- expired mapping rejected;
- closed session mapping removed;
- restart invalidates mapping;
- concurrent cancels are idempotent;
- audit sequence is complete.

## Current Known Weaknesses

- Memory-only mappings break cancel after AXIS restart.
- Persisted mappings require secret protection.
- Cancel outcome may still be timing-dependent.

## Success Looks Like

Clients can cancel long-running queries through AXIS without learning backend cancellation secrets.

## Failure Looks Like

AXIS passes through BackendKeyData, then calls itself a security control while handing out backend cancel keys like party favors.
