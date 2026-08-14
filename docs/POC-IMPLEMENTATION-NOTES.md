# Native PostgreSQL Wire Simple Query Laboratory PoC Notes

## What Was Implemented

- Disabled-by-default AXIS PG wire lab listener.
- Cleartext PostgreSQL startup/authentication pass-through in lab mode.
- Simple Query (`Q`) interception only.
- Transport-neutral PG wire request envelope mapped into the existing AXIS policy model.
- Existing audit WAL logging for deterministic PG wire decision events.
- Existing approval store ticket creation for `REQUIRE_APPROVAL`.
- Original frontend Simple Query bytes forwarded only for `ALLOW`.
- PostgreSQL-compatible AXIS `ErrorResponse` messages for blocked, approval-required, unsupported, backend-failed, policy-failed, and audit-failed paths.
- Byte-level backend mock tests proving backend bytes for ALLOW and zero original query bytes for BLOCK, approval, unsupported protocol, audit failure, policy failure, and backend failure.
- Strict transaction behavior for local reject inside a transaction: mark poisoned, issue audited AXIS-generated policy-exempt `ROLLBACK`, and close.

## What Was Not Implemented

- Extended Query Protocol: Parse, Bind, Describe, Execute, Sync, Flush, Close.
- Prepared statement or portal state.
- COPY protocol or COPY stream handling.
- TLS termination, SCRAM verification, or production authentication.
- Production CancelRequest mapping.
- Connection pooling, PgBouncer behavior, load balancing, HA, or multi-AXIS consistency.
- Query rewriting, SQL comment injection, result rewriting, or ORM compatibility.

Unsupported protocol paths fail closed. This is not production-ready and not enterprise pilot-ready.

## Crate / Dependency Decision

No new dependency was added.

The repository already has `sqlparser` for SQL classification and `tokio` for async TCP. For the laboratory scope, the PG wire code uses a narrow local decoder limited to:

- startup packet length framing;
- explicit SSL/GSS/Cancel/startup code detection;
- frontend length-prefixed message framing;
- Simple Query SQL extraction from a null-terminated `Q` payload;
- backend length-prefixed response relay;
- AXIS-generated ErrorResponse and ReadyForQuery encoding.

This is not a full PostgreSQL protocol parser and is not documented as production protocol support.

## Lab Port / Config

Defaults:

```text
AXIS_PGWIRE_ENABLED=false
AXIS_PGWIRE_LISTEN_ADDR=0.0.0.0:6544
AXIS_PGWIRE_BACKEND_ADDR=127.0.0.1:5432
AXIS_PGWIRE_LAB_MODE=true
AXIS_PGWIRE_UNSUPPORTED_FAIL_CLOSED=true
```

The existing HTTP gate listener remains unchanged.

## Run Locally

Start AXIS with PG wire explicitly enabled and point it at a backend PostgreSQL server:

```powershell
$env:AXIS_PGWIRE_ENABLED="true"
$env:AXIS_PGWIRE_BACKEND_ADDR="127.0.0.1:5432"
cargo run
```

Use a cleartext/simple-query-capable client. For `psql`, disable SSL if needed:

```powershell
psql "host=127.0.0.1 port=6544 dbname=prod_main user=varux sslmode=disable"
```

## Local Lab Audit WAL Remediation

AXIS must not run protected query execution when audit WAL integrity is broken. If startup or PG wire smoke fails with `audit_storage_corrupt`, `audit_wal_corrupt`, or an audit hash continuation mismatch, treat the audit evidence as unhealthy. Do not bypass audit and do not delete evidence from any shared, reviewer, pilot, or production environment.

For an intentional fresh local lab smoke only:

1. Stop AXIS.
2. Back up the current local audit files if they may contain useful evidence:

```powershell
Copy-Item .\audit.wal .\audit.wal.lab-backup -ErrorAction SilentlyContinue
Copy-Item .\audit.log .\audit.log.lab-backup -ErrorAction SilentlyContinue
Copy-Item .\data\audit.wal .\data\audit.wal.lab-backup -ErrorAction SilentlyContinue
Copy-Item .\data\audit.log .\data\audit.log.lab-backup -ErrorAction SilentlyContinue
```

3. Delete or reset only the audit files used by that local lab run, for example `.\audit.wal` and `.\audit.log`, or `.\data\audit.wal` and `.\data\audit.log` if the lab env points there.
4. Restart AXIS so the audit logger starts a new genesis chain.
5. Rerun the direct backend `psql` smoke and then the AXIS PG wire `psql` smoke.

The WAL is the source of truth. The JSONL projection is operator convenience and may be regenerated or reset in local lab mode, but projection reset does not repair a corrupt WAL chain.

## Run Tests

```powershell
cargo test pgwire
cargo test
```

## Known Limitations

- Startup/auth pass-through does not verify identity. Observed database user is stored only as `unauthenticated_claimed_db_user`.
- BackendKeyData is passed through in lab mode; production CancelRequest behavior is intentionally not implemented.
- Modern drivers and ORMs commonly use Extended Query and are intentionally unsupported here.
- SQL classification is limited to the existing AXIS classifier and fails closed on unsupported or ambiguous SQL.
- The local decoder enforces message limits but does not provide broad PostgreSQL protocol compatibility.
- The transaction reset path is strict and may close connections after BLOCK or approval-required decisions inside transactions.

## Explicit Posture

This is a Native PostgreSQL Wire Simple Query laboratory proof of concept only. It is not production-ready, not enterprise pilot-ready, not ORM-compatible, not a complete PostgreSQL proxy, and not a PgBouncer replacement.
