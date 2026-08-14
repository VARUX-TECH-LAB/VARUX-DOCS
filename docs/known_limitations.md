# AXIS Known Limitations

Status date: 2026-07-05

## Stored Procedure and Function Bodies Are Opaque

AXIS policies the SQL text sent by the client. It cannot see or policy the body
of routines that were previously created with `CREATE FUNCTION` or
`CREATE PROCEDURE`.

This is an architectural limit for any proxy operating at the PostgreSQL wire
protocol layer. It is not specific to AXIS.

Example: if a client sends `SELECT process_order(123)` and the
`process_order` function body contains `DELETE FROM orders`, AXIS sees only
the function call. It does not see the internal `DELETE`.

Recommended complementary control: restrict function and procedure creation and
execution with PostgreSQL native `GRANT`/`REVOKE`. For example, grant `EXECUTE`
only to trusted roles and remove `CREATE FUNCTION` privileges from application
users.

Regression coverage: `test_function_call_body_is_opaque_to_policy_documented_limitation`
creates a function whose body deletes from `orders`, calls it through AXIS, and
verifies the body is treated as opaque to policy.

## COPY Protocol Is Not Supported

AXIS does not support PostgreSQL `COPY FROM STDIN`, `COPY TO STDOUT`, or the
CopyData sub-protocol in pgwire mode. COPY traffic is fail-closed: AXIS rejects
it before entering bulk data transfer, keeps COPY data from passing silently to
PostgreSQL, and leaves the connection usable after a clean rejection.

## Sensitive Session GUCs Are Denied

AXIS allows only a narrow whitelist of PostgreSQL session parameters whose
effect is cosmetic or session-local and does not change policy classification,
object resolution, or execution identity. `search_path`, `role`,
`session_authorization`, and `row_security` are permanently denied in pgwire
mode unless a future, separate security review proves a safe design.

Adding any of those GUCs to the whitelist requires its own empirical bypass
analysis and approval. `search_path` is especially sensitive because it changes
how unqualified table and function names resolve inside PostgreSQL.
