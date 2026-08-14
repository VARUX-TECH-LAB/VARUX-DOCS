---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# Transaction State Model

## Purpose

This document defines how AXIS tracks, constrains, and recovers PostgreSQL transaction state in Native PG mode.

Transaction behavior is the largest correctness risk in AXIS Native PG mode. If client, AXIS, and backend PostgreSQL disagree about transaction state, enforcement and audit claims become unreliable.

## Scope

This document covers:

- transaction control command classification;
- strict and future lenient transaction behavior;
- BLOCK and APPROVAL_REQUIRED inside and outside transactions;
- AXIS-issued safety ROLLBACK behavior;
- ReadyForQuery transaction status handling;
- connection poisoning;
- commands such as DISCARD ALL, RESET ALL, SAVEPOINT, and client ROLLBACK;
- retry storm risks and application expectations.

## Non-Goals

- Implementing full savepoint continuation in the Simple Query PoC.
- Guaranteeing zero application disruption for blocked in-transaction writes.
- Providing a distributed transaction coordinator.
- Making dangerous transactions pleasant. Security controls are rarely spa treatments.

## Core Risk: Transaction Divergence

Divergence means the parties disagree:

| Party | Example Belief |
|---|---|
| Client | Transaction is active and can be committed |
| AXIS | Transaction was poisoned by a blocked statement |
| Backend | Transaction is active with only some statements executed |

This is not cosmetic. A blocked statement inside a transaction may leave earlier allowed statements pending. If the client later commits, those earlier statements become durable even though the transaction contained a policy violation.

## State Sources

AXIS must combine:

- SQL classification of transaction commands;
- backend `ReadyForQuery` transaction status byte;
- internal session state;
- audit intent/completion records;
- failure events;
- connection lifecycle events.

Backend `ReadyForQuery` remains the strongest source for backend transaction state after forwarded queries. AXIS-generated `ReadyForQuery` must be used only when backend was not touched or after AXIS has explicitly reset/closed the session.

## Transaction States

| State | Meaning | Reuse Allowed |
|---|---|---|
| Idle | No active transaction | Yes |
| InTransaction | Active backend transaction | Conditional |
| FailedTransaction | Backend transaction error state | Only rollback/reset |
| PoisonedByAXIS | AXIS blocked or interrupted operation and cannot safely continue | No |
| ResetAttempted | AXIS issued safety ROLLBACK/reset | No until confirmed |
| ResetConfirmed | Backend confirmed rollback/reset | Future optional reuse |
| ExecutionUnknown | Backend execution status unknown | No |
| Closed | Connection closed | No |

## ReadyForQuery Status Byte Rules

PostgreSQL `ReadyForQuery` carries transaction status:

| Byte | Meaning |
|---|---|
| `I` | Idle |
| `T` | In transaction |
| `E` | Failed transaction |

AXIS must never emit a misleading status byte.

### BLOCK outside transaction

Backend is Idle and query is not forwarded:

- AXIS may emit ErrorResponse followed by ReadyForQuery `I`.
- Connection may remain open.

### APPROVAL_REQUIRED outside transaction

Backend is Idle and query is not forwarded:

- AXIS may emit ErrorResponse with structured approval detail followed by ReadyForQuery `I`.
- Connection may remain open.

### BLOCK inside transaction

AXIS must not emit `I` unless backend reset was confirmed. v1.2 strict default is:

1. Do not forward blocked query.
2. Return ErrorResponse if possible.
3. Issue policy-exempt safety ROLLBACK to backend.
4. Audit rollback attempt and result.
5. Close client connection.

If rollback is confirmed before close, audit records `ResetConfirmed`. If rollback is not confirmed, audit records `ExecutionUnknown` or `TransactionResetUnknown` depending on dispatch state.

### APPROVAL_REQUIRED inside transaction

Approval cannot wait while locks remain active. v1.2 strict default:

1. Create or reuse approval ticket.
2. Do not forward original query.
3. Issue safety ROLLBACK.
4. Return ErrorResponse if possible.
5. Close connection.
6. Audit `approval_required_in_transaction=true` and `transaction_reset_attempted=true`.

## Policy Treatment of Transaction Control Commands

| Command | v1.2 Handling | Notes |
|---|---|---|
| BEGIN | Policy-evaluated, usually allow | Starts AXIS transaction tracking |
| START TRANSACTION | Same as BEGIN | Must normalize synonyms |
| COMMIT | Policy-evaluated unless session poisoned | Poisoned sessions must not commit |
| END | Same as COMMIT | Alias |
| Client ROLLBACK | Always allowed unless protocol invalid | Must reset AXIS state after backend confirms |
| AXIS safety ROLLBACK | Policy-exempt, audit-recorded | Safety primitive, not user operation |
| SAVEPOINT | Unsupported in Simple Query PoC or policy-blocked | Required for future lenient mode |
| ROLLBACK TO SAVEPOINT | Unsupported in Simple Query PoC or policy-blocked | Future lenient support |
| RELEASE SAVEPOINT | Unsupported in Simple Query PoC or policy-blocked | Future lenient support |
| DISCARD ALL | Block by default | Can reset state outside AXIS expectations |
| RESET ALL | Block or evaluate against protected GUC list | May clear identity/correlation context |
| SET | Evaluated; protected GUCs blocked | Includes search_path, role, timeout, AXIS metadata |
| SET ROLE | Block by default | Identity ambiguity |
| SET SESSION AUTHORIZATION | Block by default | Identity bypass risk |

## Safety ROLLBACK Exception

When AXIS issues ROLLBACK to recover from BLOCK or APPROVAL_REQUIRED inside an active transaction, that ROLLBACK bypasses normal policy evaluation.

This exception must be explicit, documented, and auditable:

```json
{
  "event_type": "AXIS_SAFETY_ROLLBACK_ISSUED",
  "policy_exempt": true,
  "reason": "transaction_poisoned_by_block_or_approval",
  "axis_request_id": "req_...",
  "session_id": "sess_...",
  "backend_session_id": "backend_...",
  "original_decision": "BLOCK",
  "rollback_dispatch_intent": true
}
```

Rollback response must produce one of:

| Result | Meaning |
|---|---|
| `rollback_confirmed` | Backend acknowledged reset |
| `rollback_failed` | Backend returned error |
| `rollback_unknown` | Connection failed before confirmation |
| `rollback_not_attempted` | Backend was unreachable or state unknown |

## Strict Mode

Strict mode is the v1.2 default.

Behavior:

- Outside transaction: BLOCK/APPROVAL_REQUIRED rejects only the statement and may keep connection open.
- Inside transaction: BLOCK/APPROVAL_REQUIRED poisons the session, attempts rollback, audits, and closes.
- Unsupported protocol inside transaction poisons and closes.
- Execution unknown closes.

Why strict is default:

- It avoids false confidence about transaction state.
- It prevents COMMIT after a policy violation.
- It is easier to audit.
- It is harsh but coherent. Coherence wins.

## Lenient Mode: Future Experimental Design

Lenient mode is not part of Simple Query PoC and must not be enabled in production until specifically implemented and tested.

A future lenient mode may use AXIS-managed savepoints:

1. Before a risky candidate statement, AXIS creates a savepoint.
2. If policy BLOCKs, AXIS rolls back to that savepoint.
3. Transaction may continue.
4. Audit marks `partial_transaction_continued=true`.
5. Policy must explicitly allow lenient continuation.

Hard requirements:

- AXIS must own savepoint names and prevent client collision.
- Savepoint commands must be hidden from client or reconciled correctly.
- Backend and client ReadyForQuery semantics must remain valid.
- Driver behavior must be tested.
- `COMMIT` after blocked statement must be policy-controlled.
- Audit must record preceding statements hash and continuation risk.

Known concern: lenient mode can create a dangerous illusion that a partially blocked business transaction is still semantically valid. Databases cannot infer business meaning. Amazing that this has to be said, but here we are.

## Retry Storm Risk

Strict rollback+close may trigger application retry storms. Mitigation:

- ErrorResponse must carry machine-readable AXIS reason.
- Application integration guide must recommend bounded retries.
- AXIS may rate-limit repeated risky queries.
- Approval tickets must deduplicate repeated attempts.
- Connection pools must discard poisoned connections.
- Operators must observe `transaction_poisoned_count` and `retry_after_block_count`.

## Application Contract

Applications must treat the following as non-retryable without human/operator logic:

- AXIS policy BLOCK.
- AXIS approval required.
- AXIS transaction poisoned.
- AXIS execution unknown.

Applications may retry only after:

- approval was granted;
- transaction was rebuilt from the beginning;
- the operation is idempotent;
- business logic permits retry.

## Current Known Weaknesses

- Strict mode may be operationally disruptive for long-lived transactions.
- Lenient mode is not yet implemented.
- Savepoint semantics are complex and driver-dependent.
- Rollback confirmation may be lost during crash.
- Transaction boundaries hidden behind PgBouncer transaction pooling are unsupported.

## Success Looks Like

AXIS never allows a client to commit a transaction after AXIS knows that transaction was poisoned by a blocked or approval-required operation.

## Failure Looks Like

AXIS blocks a dangerous statement, leaves earlier statements pending, allows COMMIT, and then writes a confident audit event as if nothing weird happened. That would be security theater with a Rust compiler.

## Acceptance Criteria

- Every transaction state transition is auditable.
- Safety ROLLBACK is explicitly policy-exempt and logged.
- BLOCK/APPROVAL inside transaction cannot silently continue.
- Client-visible ErrorResponse behavior is deterministic.
- Connection pool discard guidance exists.
- Retry storm metrics exist.
