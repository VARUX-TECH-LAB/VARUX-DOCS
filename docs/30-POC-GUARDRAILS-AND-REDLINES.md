# 30 — PoC Guardrails and Red Lines

## Purpose

This document defines the non-negotiable guardrails for the first AXIS Native PostgreSQL Wire Protocol laboratory PoC.

This PoC is not an enterprise pilot.
This PoC is not production-ready.
This PoC is not a general PostgreSQL proxy.
This PoC exists only to prove that AXIS can sit in the PostgreSQL wire path, intercept Simple Query traffic, evaluate policy before execution, enforce ALLOW/BLOCK/APPROVAL_REQUIRED decisions, and preserve audit evidence.

## Scope

The PoC scope is intentionally narrow:

- Accept a PostgreSQL client connection.
- Pass through startup/authentication flow in lab mode.
- Detect Simple Query messages.
- Extract SQL from Simple Query messages.
- Build an AXIS request envelope.
- Evaluate the SQL through the existing AXIS policy engine.
- Forward ALLOW queries to the backend PostgreSQL instance.
- Return PostgreSQL-compatible ErrorResponse for BLOCK decisions.
- Return PostgreSQL-compatible ErrorResponse for APPROVAL_REQUIRED decisions.
- Ensure BLOCK and APPROVAL_REQUIRED queries are never forwarded to backend PostgreSQL.
- Emit ordered audit events for every critical decision.
- Preserve fail-safe behavior under uncertainty.

## Non-Goals

The PoC must not implement:

- Extended Query Protocol.
- Parse / Bind / Execute / Sync support.
- Prepared statement state machine.
- Portals.
- COPY protocol.
- COPY FROM STDIN.
- COPY FROM PROGRAM.
- LISTEN / NOTIFY.
- Pipeline mode.
- Large objects.
- Replication protocol.
- CancelRequest production behavior.
- TLS termination.
- SCRAM production authentication.
- Connection pooling.
- PgBouncer replacement behavior.
- Load balancing.
- Query rewriting.
- SQL comment injection.
- Production HA.
- Multi-AXIS consistency.
- Approval store HA.
- Enterprise observability.

Unsupported protocol paths must fail closed.

## Red Lines

### 1. No monolithic proxy implementation

Do not place all logic inside a single `proxy.rs`.

The implementation must separate responsibilities:

- listener
- session lifecycle
- protocol message handling
- SQL interception
- backend forwarding
- client response generation
- policy envelope construction
- audit event emission
- state tracking

A monolithic proxy implementation is rejected.

### 2. No custom PostgreSQL protocol from scratch unless explicitly justified

Do not invent a PostgreSQL wire protocol parser from scratch.

Use a reviewed crate or a narrow, documented protocol decoder for the limited PoC scope.

If a crate is used, document:

- crate name
- version
- why it was selected
- what it handles
- what AXIS still handles manually
- known limitations

### 3. BLOCK must never reach backend

For BLOCK decisions:

- The original query must not be forwarded.
- No query byte from the blocked SQL may be written to the backend socket.
- Audit must record `backend_forwarded=false`.
- Audit must record `tcp_bytes_forwarded=0`.
- Client must receive PostgreSQL-compatible ErrorResponse.

PostgreSQL backend logs are not sufficient proof.

A byte-level backend mock or equivalent socket-level test must prove that blocked queries never reach backend.

### 4. APPROVAL_REQUIRED must never wait on the TCP socket

For APPROVAL_REQUIRED decisions:

- Do not keep the client socket waiting for human approval.
- Do not hold backend transaction locks.
- Do not forward the query to backend.
- Create approval ticket.
- Emit audit evidence.
- Return PostgreSQL-compatible ErrorResponse with machine-readable approval context.

Approval is out-of-band.

### 5. Unsupported protocol messages fail closed

The PoC must fail closed for unsupported protocol paths, including:

- Extended Query messages
- COPY
- FunctionCall
- replication startup
- pipeline mode
- unsupported CancelRequest behavior
- unknown message types

Fail closed means:

- do not forward unsafe or unknown traffic
- emit audit event
- return controlled PostgreSQL-compatible error where possible
- close connection if state becomes unsafe

### 6. Startup/auth pass-through is lab-only

Startup/auth pass-through in the PoC does not mean AXIS has verified client identity.

Any identity field derived from startup parameters must be treated as claimed and unauthenticated unless AXIS explicitly verifies it.

Policy enforcement must not rely on unauthenticated identity for high-trust decisions.

### 7. No query rewriting

AXIS must not rewrite user SQL.

ALLOW path must forward original query bytes.

Any AXIS-generated metadata command must be explicitly documented and must not silently alter user transaction semantics.

### 8. Transaction state must be explicit

AXIS must track at least:

- Idle
- InTransaction
- FailedTransaction
- PoisonedByAXIS
- ResetAttempted
- ResetConfirmed
- ExecutionUnknown
- Closed

If AXIS cannot determine safe state, it must fail closed.

### 9. Safety ROLLBACK is AXIS-generated and audit-recorded

If AXIS issues a safety ROLLBACK, it must be:

- explicitly AXIS-generated
- policy-exempt
- audit-recorded
- clearly distinguishable from client-generated ROLLBACK

### 10. Audit ordering must be deterministic

Audit events must follow a stable order:

1. query received
2. policy evaluated
3. decision applied
4. backend dispatch intent, only for ALLOW
5. backend dispatched, only after actual socket write
6. backend completed / failed / unknown
7. client response emitted

BLOCK and APPROVAL_REQUIRED must not emit backend dispatched events.

### 11. ErrorResponse must be PostgreSQL-compatible

AXIS-generated errors must include at least:

- Severity
- SQLSTATE
- Message
- Detail
- Hint where useful
- AXIS request ID
- approval ticket ID where applicable

ErrorResponse behavior must be tested against real PostgreSQL clients where possible.

### 12. Byte-level tests are mandatory

The PoC must include tests proving:

- ALLOW reaches backend.
- BLOCK does not reach backend.
- APPROVAL_REQUIRED does not reach backend.
- Unsupported protocol paths do not reach backend.
- Audit events are emitted in correct order.
- Backend down fails closed.
- Audit unavailable fails closed.
- Policy unavailable fails closed.

### 13. Existing AXIS guarantees must not regress

The PoC must not weaken:

- policy-before-execution
- audit evidence
- audit hash-chain continuity
- policy manifest integrity
- approval immutability
- fail-safe behavior
- chaos/failure-mode protections

### 14. This PoC cannot be marketed as production-ready

This PoC may only be described as:

“Native PostgreSQL Wire Simple Query laboratory proof of concept.”

It must not be described as:

- production-ready
- enterprise-ready
- drop-in replacement
- ORM-compatible
- complete PostgreSQL proxy
- PgBouncer alternative

## Acceptance Criteria

The PoC is accepted only if:

- psql or a controlled Simple Query client can connect.
- Safe Simple Query SELECT can pass through AXIS and return backend result.
- Risky write can be BLOCKED before backend dispatch.
- APPROVAL_REQUIRED creates ticket and returns controlled ErrorResponse.
- Byte-level backend mock proves zero backend bytes for BLOCK and APPROVAL_REQUIRED.
- Audit events include policy metadata.
- Unsupported protocol paths fail closed.
- Existing regression and chaos tests still pass.
- No production claims are introduced.

## Final Rule

If the implementation is unsure, it must not forward.

Uncertainty means BLOCK, audit, and controlled failure.