# 31 - Simple Query PoC Results

## Purpose

This document records the completed AXIS Native PostgreSQL Wire Simple Query laboratory proof-of-concept result.

It is an evidence summary for the Simple Query lab PoC only. It does not expand the supported protocol surface and does not create a production or pilot readiness claim.

## Scope

This result applies only to the AXIS Native PostgreSQL Wire Simple Query laboratory PoC.

Explicit scope boundaries:

- Simple Query lab PoC only.
- Not production-ready.
- Not enterprise pilot-ready.
- Not ORM-compatible.

## Environment

Validated local environment:

- Windows local repo path: `C:\FOR S3LOC\AXIS-main`
- Backend PostgreSQL via `docker compose postgres`
- Backend DB: `prod_main`
- Backend user: `varux`
- Backend password: `varux`
- AXIS PG wire listener: `6544`
- HTTP gate listener: `6543`
- PG wire mode enabled via environment variables.

## Commands Validated

The following command paths were validated during the lab result:

- `docker compose up -d postgres`
- Direct backend `psql` `SELECT 1`
- AXIS PG wire `psql` `SELECT 1`
- Risky write through AXIS
- `/audit/verify`

## Results

Observed lab results:

- Direct backend `SELECT 1` passed.
- AXIS PG wire `SELECT 1` passed.
- Risky write was blocked with a PostgreSQL `ErrorResponse`.
- Risky write did not reach the backend.
- WAL evidence included `source=pgwire`, `query_mode=simple`, `decision=BLOCK`, `backend_forwarded=false`, and `tcp_bytes_forwarded=0`.
- `/audit/verify` returned `verified`.

## Test Results

Validated test results:

- `cargo fmt` passed.
- `cargo check` passed with existing dead-code warnings.
- `cargo test` passed.
- `cargo test pgwire` passed.
- `axis_regression.py` passed, `65/65`.
- `axis_chaos_test.py` passed, `14/14`.
- `axis_gate_stress.py` passed.
- `scripts/verify_pilot_evidence.py` passed.

## Root Cause Fixed

The previous smoke failure was caused by an audit WAL hash continuation mismatch during PG wire startup.

Resolution notes:

- WAL source of truth was verified clean.
- Historical JSONL projection mismatch was not treated as source of truth.
- PG wire now uses the same `AuditLogger` path.
- Audit corruption or audit unavailability still fails closed.
- PG wire startup returns a controlled PostgreSQL `ErrorResponse` where possible if audit evidence cannot be written.

## Security Guarantees Preserved

The lab result preserved the existing AXIS security posture:

- No audit bypass.
- No fail-open.
- No protected query execution when audit storage is unhealthy.
- `BLOCK` and `APPROVAL_REQUIRED` send zero original query bytes to the backend.
- Existing audit verification still passes.

## Known Limitations

The following remain unsupported and outside the completed Simple Query lab PoC:

- No Extended Query.
- No prepared statements.
- No `COPY`.
- No TLS termination.
- No SCRAM verification.
- No production `CancelRequest`.
- No pooling.
- No PgBouncer behavior.
- No HA.
- No ORM compatibility claim.

## Next Technical Gate

The next technical gate remains separate from this lab result and requires additional design, implementation, and validation work:

- Extended Query State Machine RFC/spec.
- Parser equivalence test suite.
- Approval store HA selection.
- `CancelRequest` production implementation.
- Operator runbook and observability.
- Production pilot readiness checklist.
