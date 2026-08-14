---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# Test Matrix

## Purpose

This document defines the test matrix required before AXIS Native PG mode can progress from lab PoC to production pilot candidate.

The test matrix must prove not only that good-path traffic works, but also that dangerous traffic does not reach backend PostgreSQL and that uncertainty is surfaced honestly.

## Test Levels

| Level | Purpose |
|---|---|
| Unit | Parser, classifier, error builder, envelope mapping |
| State Machine | Session/transaction transitions without real sockets |
| Byte-Level Backend Mock | Prove backend bytes and ordering |
| PostgreSQL Integration | Validate behavior against real PostgreSQL |
| Driver Compatibility | Validate target clients and ORMs |
| Chaos | Network, crash, disk, timeout, partial write |
| Operability | Metrics, logs, traces, dashboards, runbooks |
| Security Regression | Bypass corpus |

## Mandatory Byte-Level Backend Mock

PostgreSQL logs are not enough to prove non-forwarding. Logging configuration can be wrong, and absence in logs is not proof of absence on the wire. Humanity learned this and still writes log-based tests. Delightful.

AXIS must have a backend mock that records exact bytes received from AXIS.

Acceptance criteria:

- BLOCK path writes zero original query bytes to backend mock.
- APPROVAL_REQUIRED path writes zero original query bytes to backend mock.
- Unsupported COPY path writes zero COPY payload bytes to backend mock.
- Extended Query unsupported path does not forward Parse/Bind/Execute bytes in PoC.
- Safety ROLLBACK bytes are distinguishable from user query bytes and audit-recorded.
- ALLOW path forwards original query bytes unchanged.

## Core Simple Query PoC Tests

| Test | Expected Result |
|---|---|
| psql connects through AXIS | Startup/auth pass-through succeeds |
| SELECT 1 | ALLOW and backend response pass-through |
| UPDATE safe scoped row | Policy-dependent ALLOW or APPROVAL |
| DELETE without WHERE | BLOCK before backend |
| DROP TABLE | BLOCK before backend |
| Multi-statement `SELECT 1; DROP TABLE x;` | BLOCK entire message |
| COPY table FROM STDIN | BLOCK before COPY stream |
| COPY table FROM PROGRAM | BLOCK as SQL-level dangerous command |
| SELECT pg_sleep(100) | Policy-dependent block/rate limit |
| SELECT function_with_side_effect() | BLOCK unless allowlisted |

## Transaction Tests

| Scenario | Expected Result |
|---|---|
| BEGIN; safe INSERT; COMMIT | ALLOW path, audit transaction opened/closed |
| BEGIN; safe INSERT; dangerous DELETE | BLOCK, safety ROLLBACK attempt, connection close |
| BEGIN; dangerous DDL requiring approval | approval ticket, rollback attempt, close |
| BEGIN; client ROLLBACK | always allowed, state reset after backend confirms |
| Poisoned connection sends COMMIT | reject/close, never forward COMMIT |
| DISCARD ALL | blocked by default |
| RESET ALL | blocked/evaluated against protected GUCs |
| SET search_path | blocked or policy-gated |
| SET statement_timeout | blocked or protected per config |
| SAVEPOINT commands | unsupported in PoC unless lenient mode test branch |

## Extended Query Tests

Extended Query is not part of Simple Query PoC, but rejection behavior must be tested.

| Message | PoC Expected Result |
|---|---|
| Parse | FeatureNotSupported ErrorResponse, no backend forward |
| Bind | FeatureNotSupported ErrorResponse, no backend forward |
| Execute | FeatureNotSupported ErrorResponse, no backend forward |
| Sync | handled only to keep client from hanging if safe |
| Describe | FeatureNotSupported unless part of clean rejection |

Production pilot cannot start until separate Extended Query positive tests exist for target drivers.

## Protocol Boundary Tests

| Feature | Test |
|---|---|
| SSLRequest | PoC returns documented denial or handles according to TLS mode |
| CancelRequest | PoC logs unsupported; production candidate must support mapping |
| BackendKeyData | AXIS must not leak real backend key in production candidate |
| ReadyForQuery | Correct transaction byte behavior |
| ErrorResponse | Required fields S,V,C,M,D,H present |
| NoticeResponse | Pass-through from backend |
| ParameterStatus | Pass-through or controlled override |
| Pipeline mode | rejected or disabled; no undefined behavior |
| Large object functions | blocked unless policy allowlisted |
| LISTEN/NOTIFY | unsupported or pass-through with explicit matrix behavior |

## Driver Compatibility Matrix

Minimum lab:

- psql.
- psycopg2 simple mode.
- node-postgres simple query path.

Pre-production candidate:

- psycopg3.
- asyncpg.
- Prisma.
- Java JDBC/Hibernate.
- Npgsql.
- Go pgx.

For each driver record:

- protocol mode;
- TLS behavior;
- prepared statement caching;
- error mapping;
- connection pool behavior after AXIS block;
- cancel behavior;
- retry behavior;
- approval ErrorResponse parsing.

## PgBouncer Tests

| Mode | v1.2 Status |
|---|---|
| No PgBouncer | Lab PoC supported |
| PgBouncer session pooling after AXIS | Candidate with tests |
| PgBouncer transaction pooling | Unsupported |
| PgBouncer before AXIS | Not recommended / requires explicit review |

Required tests for session pooling:

- `server_reset_query` behavior.
- `DISCARD ALL` / `RESET ALL` visibility.
- prepared statement cleanup.
- backend session reuse.
- application_name/correlation preservation.

## Failure and Chaos Tests

| Failure | Expected Result |
|---|---|
| Backend down before query | ErrorResponse, no false ALLOW |
| Backend timeout after forward | execution_unknown or backend_dispatch_unknown |
| Client disconnect before backend response | client_delivery_unknown or execution_unknown depending backend state |
| AXIS crash after dispatch intent | restart recovery marks incomplete |
| AXIS crash after backend completed before client delivery | backend_confirmed_client_delivery_unknown |
| Audit WAL unavailable | readiness fail and write operations blocked |
| Audit disk full | circuit breaker opens |
| Policy engine unavailable | fail closed |
| Policy manifest invalid | fail closed |
| Approval store down | approval policies fail closed or reject according to policy |
| Shared approval store split-brain | prevent duplicate grants or fail closed |
| Large query flood | request rejected before unbounded allocation |
| High risk query flood | rate-limited/backpressure, no OOM |

## Observability Tests

- Metrics do not use `axis_request_id`, raw SQL, raw actor IDs, or SQL hash as Prometheus labels.
- Every `axis_request_id` can be traced in logs.
- Dashboard can answer: was backend reached?
- Dashboard can answer: why was it blocked?
- Dashboard can answer: which policy version made the decision?
- Dashboard can answer: is audit healthy?
- Execution unknown events page exists.

## Security Bypass Corpus

Must include:

- comments;
- dollar quoting;
- Unicode/encoding tricks;
- multi-statement;
- CTE writes;
- SELECT INTO;
- SELECT FOR UPDATE;
- function/procedure side effects;
- COPY variants including PROGRAM;
- SET ROLE / SET SESSION AUTHORIZATION;
- search_path manipulation;
- protected GUC mutation;
- pg_sleep / DoS functions;
- large query payloads;
- prepared statements once Extended Query is implemented.

## Success Looks Like

A blocked query is proven not to reach backend at the byte level, not merely absent from PostgreSQL logs.

## Failure Looks Like

A test says “blocked query did not reach backend” because the backend log was quiet. Silence is not evidence. It is sometimes just configuration with a superiority complex.
