---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---


# PostgreSQL Protocol Boundaries

## Purpose

This document defines the PostgreSQL wire protocol boundaries for AXIS Native PG mode.

## Core Principle

AXIS must only claim support for protocol behavior it can safely observe, classify, forward, block, and audit.

Unsupported does not mean “probably fine.” Unsupported means blocked, rejected, or explicitly excluded from deployment.

## Startup Phase

### PoC Behavior

For the lab PoC, AXIS proxies startup/authentication messages between client and backend as transparently as possible.

### Important Limitation

In pass-through auth mode, AXIS does not verify the client identity. `claimed_db_user` is only the user string observed in StartupMessage and relayed through backend authentication. This must not be represented as AXIS-verified identity.

### Messages

AXIS must explicitly handle or pass through:

- StartupMessage.
- SSLRequest.
- Authentication messages.
- PasswordMessage / SASL messages.
- ParameterStatus.
- BackendKeyData.
- ReadyForQuery.
- NegotiateProtocolVersion.

## Simple Query

Simple Query is message type `Q`.

PoC supports Simple Query only for enforcement:

1. Client sends Query.
2. AXIS extracts SQL.
3. AXIS builds AxisRequestEnvelope.
4. Policy evaluates.
5. ALLOW forwards original bytes.
6. BLOCK or APPROVAL_REQUIRED does not forward query bytes.
7. AXIS returns a PostgreSQL-compatible ErrorResponse and ReadyForQuery when appropriate.

## Original Bytes Rule

For user SQL, ALLOW path must forward original query bytes. AXIS must not rewrite the SQL statement to add comments, hints, correlation IDs, or metadata.

### Correlation Exception

AXIS may issue separate session metadata commands such as `SET application_name` or safe custom GUC setup before user queries, but those are AXIS-generated control statements and must be audit-recorded separately. AXIS must not silently rewrite the user query.

## Extended Query

Extended Query includes Parse, Bind, Describe, Execute, Sync, Close, and related state.

Status: not supported in Simple Query PoC. Required before production OLTP pilot.

Reason: modern drivers and ORMs commonly use Extended Query. Supporting only Simple Query proves transport viability, not production compatibility. There, we admitted it, because denial is cheaper only until production.

## COPY

COPY must be divided into two threat classes:

1. SQL-level COPY statements, including `COPY FROM PROGRAM` and `COPY TO PROGRAM`.
2. Protocol-level COPY streams, including CopyData, CopyDone, CopyFail.

PoC behavior:

- Any COPY statement is blocked.
- `COPY FROM PROGRAM` and `COPY TO PROGRAM` are classified as critical dangerous operations.
- Any protocol COPY stream state is unsupported and must be rejected or connection-closed safely.

## CancelRequest

CancelRequest is out-of-band and uses a new connection.

PoC may mark CancelRequest unsupported, but production pilot must support AXIS-generated cancel keys and backend key mapping. Passing backend BackendKeyData directly to clients creates cancellation bypass and DoS risk.

## ReadyForQuery

ReadyForQuery contains transaction status:

- `I`: idle.
- `T`: in transaction.
- `E`: failed transaction.

AXIS must track this status from backend messages. If AXIS generates ReadyForQuery after a local BLOCK outside transaction, it must use a status consistent with AXIS/backend state.

If AXIS cannot guarantee consistency, it must close the connection and audit why.

## ErrorResponse

AXIS-generated ErrorResponse must define fields exactly:

- `S`: ERROR or FATAL.
- `V`: ERROR or FATAL.
- `C`: exact SQLSTATE.
- `M`: human-readable message.
- `D`: structured detail including `axis_request_id` and optionally `axis_ticket_id`.
- `H`: safe hint.
- Avoid misleading schema/table/column fields unless known and safe.

No “42501-style” language. Pick exact codes.

Initial recommendation:

- BLOCK: `42501` insufficient_privilege.
- APPROVAL_REQUIRED: vendor-defined structured error using `P0001` or `42501` with machine-readable Detail. Final choice must be tested against drivers.

## CommandComplete

AXIS must pass through CommandComplete but also extract row count for audit when possible:

- `INSERT 0 n`
- `UPDATE n`
- `DELETE n`
- `SELECT n`

Row count is part of enforcement evidence.

## NoticeResponse and Async Messages

NoticeResponse is non-fatal and should be passed through unless it contains unsupported protocol state.

LISTEN/NOTIFY is not supported in PoC and must be deployment-excluded.

## Pipeline Mode

Pipeline mode is unsupported. If detected or suspected, AXIS must fail closed. Request-response assumptions do not hold under pipelining.

## Large Query Handling

AXIS must enforce query size limits before unbounded buffering. A client must not be able to force AXIS to allocate a 50MB query before rejection.

## Current Known Weaknesses

- PoC pass-through auth does not verify identity.
- Simple Query-only does not cover most modern production ORM traffic.
- CancelRequest remains a production blocker until mapped.
- Protocol features must be tested at byte level, not inferred from PostgreSQL logs.

## Acceptance Criteria

A protocol feature is considered supported only if AXIS has a defined behavior for parsing, state impact, forwarding/blocking, audit evidence, and failure handling.
