---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---


# PoC Scope

## Purpose

This document defines the first native PostgreSQL PoC and prevents scope creep.

## Goal

Prove that AXIS can sit between a PostgreSQL client and backend PostgreSQL, intercept Simple Query messages, evaluate policy before forwarding, block before backend reach, and produce audit evidence.

## Non-Goal

This PoC is not production-ready and not ORM-compatible.

## Supported

- psql or controlled test client.
- Cleartext TCP.
- Startup/auth pass-through.
- ParameterStatus pass-through.
- BackendKeyData pass-through only as PoC limitation.
- ReadyForQuery pass-through and generated ReadyForQuery for local idle BLOCK.
- Simple Query `Q`.
- ALLOW forwarding of original bytes.
- BLOCK before backend forward.
- APPROVAL_REQUIRED before backend forward.
- PostgreSQL-compatible ErrorResponse.
- Audit evidence.

## Blocked or Unsupported

- Extended Query.
- COPY.
- COPY FROM PROGRAM / COPY TO PROGRAM.
- TLS termination.
- SCRAM verification.
- CancelRequest.
- LISTEN/NOTIFY.
- Pipeline mode.
- Large objects.
- Replication protocol.
- PgBouncer transaction pooling.
- Connection pooling.
- Server spoofing beyond what is necessary.

## Connection Model

One client connection maps to one backend connection.

AXIS does not pool connections in PoC.

## Query Size Limit

AXIS must enforce a maximum SQL message size before unbounded allocation. The PoC must fail closed on over-limit messages.

## Transaction Behavior

PoC strict mode:

- BLOCK outside transaction: ErrorResponse + ReadyForQuery Idle + connection remains open.
- APPROVAL outside transaction: ErrorResponse + ticket + ReadyForQuery Idle + connection remains open.
- BLOCK inside transaction: ErrorResponse if possible + AXIS safety ROLLBACK + audit + close.
- APPROVAL inside transaction: ticket + safety ROLLBACK + audit + close.

## Acceptance Tests

PoC must prove:

- Safe SELECT ALLOW reaches backend.
- Dangerous DELETE BLOCK does not reach backend.
- APPROVAL_REQUIRED does not reach backend.
- Audit logs backend_forwarded=false for blocked/approval queries.
- Backend mock verifies zero forwarded bytes for blocked query payload.
- Policy failure fails closed.
- Audit WAL failure rejects protected write.
- Backend down returns controlled error.
- Oversized query fails closed before unbounded allocation.
- COPY is blocked.
- Extended Query receives FeatureNotSupported or controlled close.
- Transaction-internal BLOCK triggers safety rollback attempt and connection close.

## Current Known Weaknesses

- BackendKeyData pass-through makes CancelRequest unsafe outside lab.
- Auth pass-through does not verify identity.
- Simple Query excludes major ORMs.
- Performance results are not enterprise-representative.

## Exit Criteria

PoC is complete only when it proves enforcement semantics, not merely connectivity. “psql connects” is a cute demo, not a product milestone.
