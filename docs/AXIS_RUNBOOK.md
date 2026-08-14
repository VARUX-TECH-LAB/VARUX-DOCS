# AXIS Runbook

AXIS v1 is not a production-ready enterprise product. It is a local hardened v1 security core. This runbook is intentionally operational and limited to the current local Docker-based baseline.

## Scope

This runbook covers:

- Starting and stopping the local runtime.
- Running the evidence baseline.
- Basic failure triage.
- Manual restore and recovery for policy, audit, approval store, and database state.

It does not cover production deployment, TLS, auth, RBAC, metrics, alerting, log rotation, distributed recovery, or enterprise incident management.

## Start

From the repo root:

```powershell
cd "C:\FOR S3LOC\AXIS-main"
docker compose up --build
```

Health check:

```powershell
Invoke-RestMethod http://localhost:6543/health
```

Expected response contains:

```text
status: ok
```

## Stop

```powershell
docker compose down
```

Do not remove volumes unless you intentionally want to reset PostgreSQL and containerized evidence data.

## Baseline Validation

After startup, run:

```powershell
python axis_regression.py --base http://localhost:6543 --fail-fast
python axis_gate_stress.py --base http://localhost:6543 --requests 1000 --concurrency 50 --approval-requests 50
python axis_audit_restart_test.py --base http://localhost:6543
python axis_chaos_test.py --base http://localhost:6543 --pool-requests 2000 --pool-concurrency 100
```

Expected current baseline:

- Regression: `RESULT total=31 passed=31 failed=0`
- Stress: unexpected result count 0
- Audit restart: `[PASS] audit chain continued across restart`
- Chaos: `CHAOS RESULT total=10 passed=10 failed=0`

## Policy Restore

When to use:

- AXIS fails startup after policy edit.
- Chaos or manual testing left `policies/prod_main.json` invalid.
- Policy behavior changed unexpectedly.

Steps:

1. Stop the runtime:

```powershell
docker compose down
```

2. Restore `policies/prod_main.json` from the last known-good copy in source control or backup.

3. Validate JSON syntax:

```powershell
python -m json.tool policies/prod_main.json
```

4. Start runtime:

```powershell
docker compose up --build
```

5. Run policy regression:

```powershell
python axis_regression.py --base http://localhost:6543 --case-file tests/policy_cases.json --fail-fast
```

6. If policy regression fails, keep AXIS out of the protected write path until the policy is repaired.

## Audit Log Recovery

When to use:

- AXIS fails startup because the final audit log entry is corrupt.
- `axis_chaos_test.py::corrupt_audit_startup_fail_fast` behavior is reproduced manually.

Current implementation detail:

- On startup, AXIS reads the final non-empty audit log entry.
- It expects a valid JSON object containing `event_hash` or legacy `hash`.
- It uses that value as the next event's `previous_hash`.

Recovery steps for local Docker data:

1. Stop runtime:

```powershell
docker compose down
```

2. Create a copy of the current container evidence volume before editing. If the container can be started for shell access, copy the file out:

```powershell
docker compose up -d postgres
docker compose run --rm --entrypoint sh dbguard -c "cp /app/data/audit.log /app/data/audit.log.recovery.bak || true"
```

3. Inspect the tail of the audit log:

```powershell
docker compose run --rm --entrypoint sh dbguard -c "tail -n 20 /app/data/audit.log"
```

4. If the final line is clearly partial or invalid JSON, remove only the corrupt trailing line. Keep the backup from step 2.

```powershell
docker compose run --rm --entrypoint sh dbguard -c 'awk "NF { last=NR } { lines[NR]=\$0 } END { for (i=1; i<last; i++) print lines[i] }" /app/data/audit.log > /app/data/audit.log.recovered && mv /app/data/audit.log.recovered /app/data/audit.log'
```

5. Start runtime:

```powershell
docker compose up --build
```

6. Verify restart continuity:

```powershell
python axis_audit_restart_test.py --base http://localhost:6543
```

Important: this is a local recovery procedure. v1 does not include full historical audit-chain verification. If evidence integrity is legally or operationally sensitive, preserve the corrupt original and escalate before editing.

## Approval Store Recovery

When to use:

- AXIS fails startup or rejects approval operations because `approvals.jsonl` contains invalid JSON.
- Pending approval state is inconsistent after manual testing.

Current implementation detail:

- Approval records are loaded from JSON lines at startup.
- Invalid approval log entries are treated as integrity errors.
- Resolved approvals cannot be resolved again.

Recovery steps:

1. Stop runtime:

```powershell
docker compose down
```

2. Backup the approval store:

```powershell
docker compose run --rm --entrypoint sh dbguard -c "cp /app/data/approvals.jsonl /app/data/approvals.jsonl.recovery.bak || true"
```

3. Inspect the tail:

```powershell
docker compose run --rm --entrypoint sh dbguard -c "tail -n 50 /app/data/approvals.jsonl"
```

4. If the last line is corrupt from an interrupted write or manual test, remove only that trailing corrupt line and retain the backup.

```powershell
docker compose run --rm --entrypoint sh dbguard -c 'awk "NF { last=NR } { lines[NR]=\$0 } END { for (i=1; i<last; i++) print lines[i] }" /app/data/approvals.jsonl > /app/data/approvals.jsonl.recovered && mv /app/data/approvals.jsonl.recovered /app/data/approvals.jsonl'
```

5. Start runtime:

```powershell
docker compose up --build
```

6. List approvals:

```powershell
Invoke-RestMethod http://localhost:6543/approvals
```

7. Reject stale pending approvals unless there is an explicit reason to approve them:

```powershell
Invoke-RestMethod http://localhost:6543/approvals/<approval_id>/resolve -Method Post -ContentType "application/json" -Body '{
  "actor": "recovery-operator",
  "decision": "reject",
  "comment": "Rejected during local recovery cleanup"
}'
```

## PostgreSQL Recovery

When to use:

- `postgres` container is down.
- `/query` read path is failing with DB connection errors.
- Chaos test stopped or restarted PostgreSQL.

Steps:

```powershell
docker compose start postgres
docker compose up -d dbguard
```

Wait for health:

```powershell
Invoke-RestMethod http://localhost:6543/health
```

Run a read smoke test:

```powershell
Invoke-RestMethod http://localhost:6543/query -Method Post -ContentType "application/json" -Body '{
  "actor": "recovery-operator",
  "app": "webapp",
  "tenant": "acme",
  "role": "admin",
  "host": "win",
  "env": "prod",
  "sql": "SELECT * FROM orders ORDER BY id"
}'
```

Then run:

```powershell
python axis_regression.py --base http://localhost:6543 --fail-fast
```

## Reset Local State

Use only when local data can be discarded.

```powershell
docker compose down -v
docker compose up --build
```

This removes PostgreSQL and containerized evidence volumes. It is not an evidence-preserving recovery path.

## Triage Checklist

For unsafe behavior concerns:

- Stop sending write traffic to AXIS.
- Preserve `audit.log`, `approvals.jsonl`, `policies/prod_main.json`, and relevant console output.
- Run parser and policy regression.
- Check whether the request received `200 ALLOW`, `202 REQUIRE_APPROVAL`, `403 BLOCK`, or controlled error.
- For any risky SQL that received `200 ALLOW`, capture the request body, response body, policy version, and audit request id.

For availability concerns:

- Check `/health`.
- Check Docker service status.
- Check PostgreSQL health.
- Check policy JSON validity.
- Check final audit log line validity.
- Check final approval store line validity.

## Known Operational Gaps

- No RBAC.
- No metrics or observability.
- No log rotation.
- Recovery procedures are manual and initial-level.
- No full historical audit-chain verifier.
- No multi-instance distributed audit chain.
- No long-running soak test.
- No production deployment hardening.
