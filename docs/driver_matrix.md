# AXIS Driver Matrix

Status date: 2026-07-05

Status update (2026-08): pipelining is enabled (`PIPELINING_SUPPORT=true`)
with a bounded batch size (`AXIS_PGWIRE_MAX_BATCH_ITEMS`). JDBC batch
execution is supported and verified end-to-end for Hibernate 6 + pgjdbc
(explicit-OID binds, multi Bind/Execute before one Sync,
savepoint/rollback) with positive and negative policy-enforcement controls
and signed audit evidence; denied batch items fail closed with connection
recovery, and oversized batches are rejected. The 2026-07-05 recon entries
below record the earlier `PIPELINING_SUPPORT=false` rejection behavior and
remain as historical observation.

This matrix tracks real-client compatibility for AXIS pgwire mode. Batch and
bulk behavior is listed separately because extended-query cycles with more
than one Bind/Execute pair before Sync were rejected while
`PIPELINING_SUPPORT=false`.

Known protocol limitations, including stored routine body opacity and COPY
fail-closed behavior, are documented in
[`docs/technical/known_limitations.md`](known_limitations.md). COPY is not supported in
pgwire mode and is rejected before AXIS enters the CopyData sub-protocol.

## Pipelining Recon

Mandatory Prisma/JDBC wire-level recon was run locally through a temporary TCP
capture proxy in front of AXIS. No Prisma or JDBC production test code was
added before this recon.

Environment:

| Check | Result |
| --- | --- |
| Docker daemon | Available: Docker server `29.4.3`. |
| PostgreSQL test container | Available through the existing real-driver harness. |
| Node.js | Available: `node --version` returned `v24.15.0`. |
| npm | Available through `npm.cmd --version`, returned `11.12.1`; PowerShell blocks `npm.ps1` under the current execution policy. |
| Java | User-local Temurin JDK installed under `C:\Users\V_LC\.axis-tools\jdk-21.0.11+10`; `java --version` returned `openjdk 21.0.11`. |
| Maven | User-local Maven installed under `C:\Users\V_LC\.axis-tools\apache-maven-3.9.11`; `mvn --version` returned `Apache Maven 3.9.11`. |

Observed recon evidence:

| Client path | Captured frontend pattern | Pipelining observed | AXIS current behavior |
| --- | --- | --- | --- |
| JDBC default connection before batch | `p/p/Q`, SQL `SET application_name = 'PostgreSQL JDBC Driver'` | No batch reached. | AXIS now allows whitelisted startup GUC `application_name`; `assumeMinServerVersion=12` is no longer required for connection startup. |
| JDBC `PreparedStatement.executeBatch()` INSERT, 3 rows, `assumeMinServerVersion=12` | `p/p/P/B/D/E/B/D/E/B/D/E/S/P/B/D/E/S` | Yes: 3 Bind and 3 Execute before one Sync. | Clean SQLSTATE `42501`, reason `unsupported_extended_pipeline`; same connection then executed `SELECT 1`. |
| JDBC `PreparedStatement.executeBatch()` INSERT, 3 rows, `reWriteBatchedInserts=true` | `p/p/P/B/D/E/P/B/D/E/S/C/P/B/D/E/S` | Yes: rewrite changed SQL shape, but still produced 2 Bind/Execute cycles before one Sync for 3 rows. | Clean SQLSTATE `42501`, reason `unsupported_extended_pipeline`; same connection then executed `SELECT 1`. |
| JDBC `PreparedStatement.executeBatch()` UPDATE, 3 rows | `p/p/P/B/D/E/B/D/E/B/D/E/S/P/B/D/E/S` | Yes: 3 Bind and 3 Execute before one Sync. | Clean SQLSTATE `42501`, policy reason `real_driver_batch_update_blocked`; same connection then executed `SELECT 1`. |
| JDBC `PreparedStatement.executeBatch()` DELETE, 3 rows | `p/p/P/B/D/E/B/D/E/B/D/E/S/P/B/D/E/S` | Yes: 3 Bind and 3 Execute before one Sync. | Clean SQLSTATE `42501`, policy reason `real_driver_delete_blocked`; same connection then executed `SELECT 1`. |
| Prisma 6.19.3 `$transaction([...])`, 3 operations | `p/p/Q/P/D/S/B/E/S/Q/P/D/S/B/E/S` | No: max 1 Bind and 1 Execute before Sync. | First UPDATE was policy-denied with SQLSTATE `42501`, Prisma rolled back, client then executed `SELECT 1`. |
| Prisma 6.19.3 interactive `$transaction(async tx => ...)`, 3 sequential operations | `p/p/Q/P/D/S/B/E/S/Q/P/D/S/B/E/S` | No: max 1 Bind and 1 Execute before Sync. | First UPDATE was policy-denied with SQLSTATE `42501`, Prisma rolled back, client then executed `SELECT 1`. |
| Prisma 6.19.3 `createMany()` with 3 rows | `p/p/Q/P/D/S/B/E/S/Q/P/D/S/B/E/S` | No: one multi-row INSERT Bind/Execute before Sync. | Succeeded, committed, client then executed `SELECT 1`. |
| Prisma 6.19.3 `updateMany()` affecting 3 rows | `p/p/Q/P/D/S/B/E/S/Q/P/D/S/B/E/S` | No: one UPDATE Bind/Execute before Sync. | Policy-denied with SQLSTATE `42501`, Prisma rolled back, client then executed `SELECT 1`. |
| Prisma 6.19.3 `deleteMany()` affecting 3 rows | `p/p/P/D/S/B/E/S/P/D/S/B/E/S` | No: one DELETE Bind/Execute before Sync. | Policy-denied with SQLSTATE `42501`, client then executed `SELECT 1`. |

Existing synthetic AXIS coverage confirms same-cycle pipelining is rejected
cleanly at the protocol state-machine level:
`extended_query_rejects_second_bind_before_sync_as_unsupported_pipeline` and
related pgwire tests assert the second parameter is not forwarded and the client
receives SQLSTATE `42501`. The real-driver captures above confirm the same clean
rejection for pgjdbc batch traffic.

Product decision (2026-07-05): **POLICY = pipelining_not_supported_document_only**.

pgjdbc batch execution (`addBatch`/`executeBatch`) was NOT SUPPORTED in MVP
and was cleanly rejected (`42501`/`unsupported_extended_pipeline`); the
connection remained usable after rejection. Regression test
`test_jdbc_batch_execute_is_cleanly_rejected_not_hung_or_corrupted` covered this.

Policy update (2026-08): pipelining is enabled with a bounded batch size
(`AXIS_PGWIRE_MAX_BATCH_ITEMS`). JDBC batch execution is supported and
verified end-to-end for Hibernate 6 + pgjdbc (explicit-OID binds, multi
Bind/Execute before one Sync, savepoint/rollback) with signed audit
evidence; denied batch items fail closed with connection recovery, and
oversized batches are rejected. See the status update at the top of this
document.

Prisma and JDBC single-statement scenarios (parameterized SELECT, INSERT,
UPDATE, denied DELETE, transaction abort, savepoint recovery) are
production-tested. JDBC default connection startup without
`assumeMinServerVersion=12` is covered by
`test_jdbc_default_connection_without_assume_min_server_version_succeeds`.

## Compatibility Matrix

| Driver / ORM | Language | Query mode observed | Parameterized SELECT | INSERT | UPDATE | Denied DELETE | Transaction abort | Savepoint recovery if applicable | Prepared statement reuse | Batch/bulk operation + pipelining behavior observed | Pool behavior | Raw value leakage check | Status | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| psycopg3 | Python | Extended Query for parameterized statements; simple SQL for explicit transaction control in tests. | Covered by `tests/test_pgwire_real_drivers.py`. | Covered. | Covered. | Covered, expects SQLSTATE `42501`. | Covered. | Covered; `test_psycopg3_savepoint_recovery_after_policy_deny` passed. | Covered with repeated prepared execution and SQL-level PREPARE/EXECUTE deny. | Not part of current psycopg3 matrix. | Basic connection reuse after deny covered. | Audit WAL marker checks included. | Implemented and locally passed. | Full real-driver suite passed 37 tests on 2026-07-05. |
| asyncpg | Python | Extended Query / asyncpg prepared statement flow. | Covered by `tests/test_pgwire_real_drivers.py`. | Covered by the raw-leakage insert path. | Covered through prepared update and transaction tests. | Covered, expects SQLSTATE `42501`. | Covered. | Covered; `test_asyncpg_savepoint_recovery_after_policy_deny` passed. | Covered with `conn.prepare()` and SQL-level PREPARE/EXECUTE deny. | Not part of current asyncpg matrix. | Connection reuse after deny covered. | Audit WAL marker checks included. | Implemented and locally passed. | Uses raw SQL transaction commands for savepoint recovery. |
| SQLAlchemy | Python | Not observed in this change. | Not added. | Not added. | Not added. | Not added. | Not added. | Not added. | Not added. | Not observed. | Not added. | Not added. | Not planned in this change. | Add only after a specific SQLAlchemy compatibility target is selected. |
| Prisma | TypeScript/JavaScript | Prisma Client 6.19.3 used Extended Query for DML and Simple Query for transaction control (`BEGIN`/`COMMIT`/`ROLLBACK`/`SAVEPOINT`). | Covered by `tests/test_pgwire_real_drivers.py`. | Covered (single-row INSERT). | Covered (single-row UPDATE). | Covered, expects SQLSTATE `42501`. | Covered. | Covered with raw SQL savepoint inside interactive `$transaction`. | Not yet covered as a production test. | Batch/bulk skipped: recon found no same-cycle pipelining. | Single PrismaClient reused for post-error `SELECT 1` in each scenario. | Production leakage test not added yet. | Single-statement production tests implemented. | Prisma Migrate / schema introspection remains out of scope; migrations should use a direct privileged connection unless product requirements change. |
| PostgreSQL JDBC | Java | pgjdbc 42.7.7 batch traffic used Extended Query; default connection may send whitelisted Simple Query `SET application_name` during startup. | Covered by `tests/test_pgwire_real_drivers.py`; default connection without `assumeMinServerVersion=12` is covered. | Covered (single-row INSERT). | Covered (single-row UPDATE). | Covered, expects SQLSTATE `42501`. | Covered. | Covered; savepoint recovery after policy deny passed. | Covered with repeated prepared execution. | Batch execution (`addBatch`/`executeBatch`): supported with bounded batch size (`AXIS_PGWIRE_MAX_BATCH_ITEMS`); verified end-to-end for Hibernate 6 + pgjdbc (explicit-OID binds, multi Bind/Execute before one Sync, savepoint/rollback) with signed audit evidence; denied items fail closed, oversized batches rejected. | Same JDBC connection executed `SELECT 1` after each recon rejection. | Production leakage test not added yet. | Single-statement production tests implemented; batch included in compatibility surface with bounded batch size (status update 2026-08). | `assumeMinServerVersion=12` is no longer required for startup. |

## CI Plan

Required jobs once the product decision gate is resolved:

| Job | Command | Notes |
| --- | --- | --- |
| Rust unit/integration | `cargo test --no-fail-fast` | Must always run. |
| Python real drivers | `python -m pytest tests/test_pgwire_real_drivers.py -vv -s` | Requires Docker daemon and PostgreSQL image access. |
| Prisma single-statement tests | `python -m pytest tests/test_pgwire_real_drivers.py -k prisma -vv -s` | Requires Docker, Node.js, and PostgreSQL image access. Prisma bind-inference coverage included per round-7 ORM ground-truth evidence. |
| JDBC single-statement tests | `python -m pytest tests/test_pgwire_real_drivers.py -k jdbc -vv -s` | Requires Docker, Java 21, and PostgreSQL image access. JDBC batch coverage (success, denied-item fail-closed recovery, size-bound rejection) included per round-8 ORM ground-truth evidence. |

JDBC batch support is green with a bounded batch size; keep the batch
success, denied-item fail-closed recovery, and size-bound rejection tests in
CI.

JDBC and Prisma single-statement test suites run in the same `real_pgwire`
fixture as psycopg3 and asyncpg. The JDBC tests invoke pgjdbc through a
subprocess Maven project at `tests/jdbc-axis-test/`. The Prisma tests use
`@prisma/client` generated against the proxy DSN.
