---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# Protocol Fidelity Matrix

## Purpose

This document classifies PostgreSQL protocol and SQL features by AXIS Native PG support level.

No feature may be described as supported unless AXIS has explicit state handling, tests, failure behavior, audit mapping, and operator visibility for that feature.

## Support Classes

| Class | Meaning |
|---|---|
| Supported in PoC | Implemented and tested in lab Simple Query PoC |
| Blocked in PoC | Explicitly rejected fail-closed |
| Required Before Pilot | Must be implemented before real OLTP pilot |
| Future | Useful but not pilot-blocking |
| Unsupported | Should not be routed through AXIS |

## Protocol Matrix

| Feature | v1.2 Class | Behavior |
|---|---|---|
| StartupMessage | Supported in PoC | Pass-through to backend |
| Authentication pass-through | Supported in PoC | AXIS does not verify identity |
| PasswordMessage / SASL messages | Supported in PoC pass-through | Must not imply AXIS verified user |
| ParameterStatus | Supported in PoC pass-through | Future controlled override for protected metadata |
| BackendKeyData | PoC pass-through; pilot blocker | Production must not leak backend key directly |
| ReadyForQuery | Supported in PoC | Must track transaction byte |
| Simple Query `Q` | Supported in PoC | Intercept/evaluate/forward or block |
| Extended Query Parse | Required Before Pilot | Not supported in Simple Query PoC |
| Extended Query Bind | Required Before Pilot | Parameter handling required |
| Extended Query Execute | Required Before Pilot | Portal/statement state required |
| Sync | Required Before Pilot | Required for Extended Query lifecycle |
| Describe | Required Before Pilot | Needed for many drivers |
| Close | Required Before Pilot | Statement/portal cleanup |
| Flush | Required Before Pilot | Driver compatibility |
| COPY SQL command | Blocked in PoC | Block all COPY statements |
| COPY FROM PROGRAM | Blocked in PoC | Critical dangerous command |
| COPY TO PROGRAM | Blocked in PoC | Critical dangerous command |
| COPY subprotocol | Blocked in PoC | No COPY stream support |
| CancelRequest | Required Before Pilot | See 26-CANCELREQUEST-DESIGN |
| SSLRequest | Blocked/unsupported in PoC | TLS roadmap required |
| GSSENCRequest | Unsupported in PoC | Future enterprise review |
| NegotiateProtocolVersion | Pass-through if observed | Must be added to tests |
| NoticeResponse | Pass-through | Non-fatal backend notice |
| ErrorResponse from backend | Pass-through | Do not rewrite backend errors |
| ErrorResponse from AXIS | Supported | Must use defined field set |
| CommandComplete | Pass-through and audit row count | Important evidence source |
| RowDescription/DataRow | Pass-through | Do not synthesize result rows |
| LISTEN/NOTIFY | Unsupported in PoC | Async message complexity |
| Pipeline mode | Unsupported | Must reject or disable |
| Large objects | Block/future review | Often function-based bypass surface |
| Replication protocol | Unsupported | Not an AXIS target |

## SQL Feature Matrix

| SQL Feature | v1.2 Class | Behavior |
|---|---|---|
| SELECT simple safe | Supported | Policy evaluated |
| INSERT/UPDATE/DELETE | Policy evaluated | Protected writes require policy |
| DDL | Block or approval | DDL in transaction risky |
| Multi-statement Simple Query | Blocked in PoC | Entire message rejected |
| CTE with write | Block/evaluate as write | Parser must detect |
| SELECT INTO | Block/evaluate as write | Writes data |
| SELECT FOR UPDATE | Risky | Locking behavior |
| Function call unknown | Block unless allowlisted | Side effects/DoS risk |
| pg_sleep | Block/rate limit | DoS risk |
| set_config | Block for protected GUCs | Metadata tampering risk |
| SET ROLE | Block | Identity confusion |
| SET SESSION AUTHORIZATION | Block | Identity bypass |
| SET search_path | Block or strict allowlist | Function resolution risk |
| SET statement_timeout | Protected | May alter AXIS expectations |
| SET lock_timeout | Protected | May alter failure behavior |
| SET idle_in_transaction_session_timeout | Protected | State recovery dependency |
| DISCARD ALL | Block | Resets session state |
| RESET ALL | Block or guarded | Clears protected context |

## COPY Detection Requirement

COPY must be detected at SQL grammar/token level, not by naive substring matching.

Invalid methods:

- `starts_with("COPY")` only;
- regex over raw SQL;
- keyword search inside comments or strings;
- assuming COPY appears only as protocol submessages.

Required method:

- parse Simple Query into PostgreSQL-compatible statement structure;
- classify top-level COPY command;
- classify COPY PROGRAM variants;
- block before backend forward;
- audit `copy_blocked=true` and `copy_variant` if known.

Until parser confidence is adequate, any ambiguous COPY-like statement must fail closed.

## ErrorResponse Field Set

AXIS-generated ErrorResponse must include:

| Field | Required | Notes |
|---|---|---|
| `S` | Yes | localized severity: ERROR |
| `V` | Yes | non-localized severity: ERROR |
| `C` | Yes | exact SQLSTATE |
| `M` | Yes | stable message |
| `D` | Yes | includes `axis_request_id`, ticket if present |
| `H` | Yes | operator/app hint |
| `P` | Optional | only if precise position known |
| `s/t/c/d/n` | No by default | do not fake schema/table/column |
| `F/L/R` | No | do not expose internal source info |

## CommandComplete Audit

For ALLOW queries, AXIS must inspect backend `CommandComplete` tag where available and audit:

- command tag;
- rows affected if parseable;
- query fingerprint;
- policy decision;
- backend completed timestamp.

Example:

```json
{
  "event_type": "AXIS_BACKEND_COMPLETED",
  "axis_request_id": "req_...",
  "command_tag": "UPDATE 50000",
  "rows_affected": 50000,
  "backend_confirmed": true
}
```

Large row counts may trigger future post-execution alerting, but they do not retroactively block an already executed query. Reality has an annoying direction called time.

## Current Known Weaknesses

- Full PostgreSQL semantic equivalence is not guaranteed.
- Extended Query remains the largest compatibility gap.
- CancelRequest requires dedicated design.
- COPY detection depends on parser quality.
- Production TLS changes protocol behavior.

## Success Looks Like

Every protocol behavior has one of: supported, blocked, required, future, or unsupported. Nothing floats in the swamp of “probably works.”
