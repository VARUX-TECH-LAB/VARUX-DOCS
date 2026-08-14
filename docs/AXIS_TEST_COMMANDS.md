# AXIS Test Commands

AXIS v1 is not a production-ready enterprise product. It is a local hardened v1 security core. The commands below reproduce the current local evidence baseline.

Run commands from the repository root:

```powershell
cd "C:\FOR S3LOC\AXIS-main"
```

## Start Runtime

```powershell
docker compose up --build
```

The local API listens on:

```text
http://localhost:6543
```

The Compose file also maps `6544:6543`, which is used by the chaos harness as a fallback if a stale local listener is detected on `6543`.

## Optional Compile Check

```powershell
cargo check
```

## Full Regression Baseline

Runs the default regression case files:

- `tests/policy_cases.json`
- `tests/parser_bypass_cases.json`
- `tests/approval_cases.json`

```powershell
python axis_regression.py --base http://localhost:6543 --fail-fast
```

Expected current baseline:

```text
RESULT total=31 passed=31 failed=0
```

## Individual Regression Suites

Policy regression:

```powershell
python axis_regression.py --base http://localhost:6543 --case-file tests/policy_cases.json --fail-fast
```

Parser bypass corpus:

```powershell
python axis_regression.py --base http://localhost:6543 --case-file tests/parser_bypass_cases.json --fail-fast
```

Approval regression:

```powershell
python axis_regression.py --base http://localhost:6543 --case-file tests/approval_cases.json --fail-fast
```

## Gate Stress Baseline

```powershell
python axis_gate_stress.py --base http://localhost:6543 --requests 1000 --concurrency 50 --approval-requests 50
```

Expected current baseline:

- Smoke/gate tests pass.
- Bypass/adversarial tests pass.
- Main stress test reports unexpected errors = 0.
- Approval stress test passes.

## Audit Restart Continuity

```powershell
python axis_audit_restart_test.py --base http://localhost:6543
```

Expected current baseline:

```text
[PASS] audit chain continued across restart
```

The test checks that the first post-restart audit event has `previous_hash` equal to the last pre-restart event hash.

## Chaos and Failure-Mode Baseline

```powershell
python axis_chaos_test.py --base http://localhost:6543 --pool-requests 2000 --pool-concurrency 100
```

Expected current baseline:

```text
CHAOS RESULT total=10 passed=10 failed=0
```

Scenarios covered:

- PostgreSQL down.
- Audit unwritable.
- Invalid policy startup fail-fast.
- Corrupt approval store fail-safe.
- Malformed JSON controlled rejection.
- Huge payload handling.
- Concurrent approval resolve race.
- Restart during traffic.
- DB pool pressure.
- Corrupt audit startup fail-fast.

## Manual Query Smoke Test

Read path:

```powershell
Invoke-RestMethod http://localhost:6543/query -Method Post -ContentType "application/json" -Body '{
  "actor": "local-dev",
  "app": "webapp",
  "tenant": "acme",
  "role": "admin",
  "host": "win",
  "env": "prod",
  "sql": "SELECT * FROM orders ORDER BY id"
}'
```

Allowed single-row update path:

```powershell
Invoke-RestMethod http://localhost:6543/query -Method Post -ContentType "application/json" -Body '{
  "actor": "local-dev",
  "app": "webapp",
  "tenant": "acme",
  "role": "admin",
  "host": "win",
  "env": "prod",
  "sql": "UPDATE orders SET status = ''manual_smoke'' WHERE id = 1"
}'
```

Approval path:

```powershell
Invoke-RestMethod http://localhost:6543/query -Method Post -ContentType "application/json" -Body '{
  "actor": "local-dev",
  "app": "webapp",
  "tenant": "acme",
  "role": "admin",
  "host": "win",
  "env": "prod",
  "sql": "UPDATE orders SET status = ''manual_bulk''"
}'
```

Block path:

```powershell
Invoke-RestMethod http://localhost:6543/query -Method Post -ContentType "application/json" -Body '{
  "actor": "local-dev",
  "app": "webapp",
  "tenant": "acme",
  "role": "admin",
  "host": "win",
  "env": "prod",
  "sql": "DELETE FROM orders"
}'
```

List approvals:

```powershell
Invoke-RestMethod http://localhost:6543/approvals
```

Resolve approval:

```powershell
Invoke-RestMethod http://localhost:6543/approvals/<approval_id>/resolve -Method Post -ContentType "application/json" -Body '{
  "actor": "dba-oncall",
  "decision": "approve",
  "comment": "Reviewed maintenance request"
}'
```

Use `"decision": "reject"` to reject an approval.

## Evidence Files

Generated runtime evidence is local:

- `audit.log` for audit events.
- `approvals.jsonl` for approval records.
- Docker volume `dbguard_data` for containerized audit and approval data.
- PostgreSQL Docker volume `pgdata` for database state.

These files can grow. v1 has no log rotation.
