---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# Codex Implementation Brief

## Purpose

This document defines the first implementation task for AXIS Native PG mode and constrains Codex or any automated coding agent from turning the repository into a PostgreSQL-proxy swamp.

The goal is not to build production support. The goal is to implement a lab-only Simple Query interception path that proves the enforcement model without corrupting the existing AXIS core.

## Non-Negotiable Scope

Implement only:

- Native PG listener.
- Startup/auth pass-through for lab mode.
- Simple Query message interception.
- `AxisRequestEnvelope` construction.
- Existing policy engine call.
- Existing audit path call.
- ALLOW forward.
- BLOCK / APPROVAL_REQUIRED no-forward response.
- PostgreSQL-compatible ErrorResponse builder.
- byte-level backend mock tests.

Do not implement:

- Extended Query support;
- TLS termination;
- SCRAM termination;
- connection pooling;
- PgBouncer compatibility logic;
- CancelRequest support except explicit unsupported handling;
- COPY stream support;
- query rewriting;
- result rewriting;
- custom SQL parser from scratch;
- global production configuration changes;
- control plane redesign.

## Required Crate Evaluation

Before coding, create a short implementation note comparing:

| Option | Use | Risk |
|---|---|---|
| `postgres-protocol` | Low-level message encoding/decoding | More manual state handling |
| `pgwire` | Higher-level server-side framework | May hide startup/auth details |
| Manual protocol subset | Only if crate cannot satisfy exact constraints | High correctness risk |

Default recommendation for PoC:

- Prefer `postgres-protocol` for message format correctness and explicit control.
- Do not write byte format encoding from scratch unless unavoidable.
- If `pgwire` is used, verify that it allows pass-through startup/auth and does not force full server spoofing.

No dependency may be added without documenting:

- license;
- maintenance status;
- protocol coverage;
- async runtime compatibility;
- security implications.

## Required Module Boundary

Do not put everything into `proxy.rs`. That is how software turns into a cursed basement.

Recommended structure:

```text
src/pgwire/
  mod.rs
  listener.rs        # TCP accept loop and config binding only
  session.rs         # per-connection lifecycle and state machine
  interceptor.rs     # message type detection and SQL extraction
  forwarder.rs       # backend socket forwarding only
  responder.rs       # ErrorResponse and ReadyForQuery generation
  envelope.rs        # AxisRequestEnvelope build
  cancel.rs          # placeholder unsupported/candidate future design
  state.rs           # PG session/transaction state model
  protocol.rs        # crate wrappers and message helpers
  errors.rs          # AXIS-to-PostgreSQL error mapping
  tests/
    backend_mock.rs
    simple_query.rs
    block_no_forward.rs
    transaction_poison.rs
```

## Data Flow

```text
client socket
  -> listener
  -> session
  -> startup/auth pass-through
  -> ReadyForQuery observed
  -> interceptor sees Simple Query
  -> envelope builder
  -> policy engine
  -> audit decision intent
  -> decision apply
       ALLOW: forward original bytes to backend
       BLOCK: do not forward; ErrorResponse + ReadyForQuery if safe
       APPROVAL_REQUIRED: ticket + audit; do not forward; ErrorResponse + ReadyForQuery if safe
  -> audit completion
```

## Original Bytes Rule

ALLOW path must forward original SQL message bytes unchanged.

Exceptions:

- AXIS-issued safety ROLLBACK is an AXIS control operation, not user SQL.
- Optional metadata correlation commands are not allowed in PoC unless explicitly added and audited as AXIS control operations.

## ErrorResponse Requirements

AXIS-generated ErrorResponse must include:

| Field | Requirement |
|---|---|
| `S` | `ERROR` |
| `V` | `ERROR` |
| `C` | Exact SQLSTATE, not “style” |
| `M` | Human-readable AXIS message |
| `D` | Structured or semi-structured details with `axis_request_id` and optional ticket |
| `H` | Stable hint for operator/application |

Default codes:

| AXIS Decision | SQLSTATE | Notes |
|---|---|---|
| BLOCK | `42501` | insufficient_privilege |
| APPROVAL_REQUIRED | `P0001` initially, pending final vendor strategy | must carry ticket ID |
| UNSUPPORTED_PROTOCOL | `0A000` | feature_not_supported |
| EXECUTION_UNKNOWN | `08006` or mapped connection failure | must not be confused with policy denial |

Do not emit table/schema/column fields unless known and correct. Lying to ORMs is a traditional enterprise pastime; avoid participating.

## Audit Requirements

Before forwarding ALLOW query:

1. `AXIS_QUERY_RECEIVED`
2. `AXIS_POLICY_EVALUATED`
3. `AXIS_BACKEND_DISPATCH_INTENT`
4. backend write
5. `AXIS_BACKEND_DISPATCHED`

For BLOCK/APPROVAL:

1. `AXIS_QUERY_RECEIVED`
2. `AXIS_POLICY_EVALUATED`
3. `AXIS_BLOCK_APPLIED` or `AXIS_APPROVAL_ISSUED`
4. include `backend_forwarded=false`
5. include `tcp_bytes_forwarded=0`

If audit append fails before protected write forward, do not forward.

## Test Requirements

Mandatory tests:

- psql simple SELECT allow.
- dangerous DELETE block.
- approval required no-forward.
- backend mock proves zero original query bytes on block.
- ErrorResponse fields validated.
- audit events ordered.
- backend down behavior.
- audit unavailable behavior.
- multi-statement block.
- COPY block.
- transaction block causes safety rollback attempt and close.

## Acceptance Criteria

The PoC is successful only if:

- safe Simple Query round-trips work;
- unsafe Simple Query never reaches backend;
- unsupported protocol fails closed;
- audit evidence is produced for each decision;
- byte-level tests prove non-forwarding;
- existing AXIS HTTP/query behavior remains unchanged;
- no monolithic proxy file is created;
- no production readiness claim is made.

## Failure Looks Like

A demo works with `psql`, someone calls it production-ready, then Prisma appears with Extended Query and the whole thing becomes a very expensive lesson in reading the PostgreSQL protocol docs.
