# AXIS Demo Flow

## Demo Objective

Show a technical reviewer that AXIS can:

- Allow a safe query.
- Block a risky query.
- Require approval for a controlled write.
- Resolve an approval.
- Inspect audit evidence.
- Verify evidence chain integrity.
- Validate, diff, and dry-run policy.
- Create a policy candidate.
- Activate a candidate.
- Roll back to a prior policy version.

## Demo Prerequisites

- Backend running on `http://localhost:6543`.
- Frontend running on `http://localhost:3000`.
- Policy loaded from `policies/prod_main.json` or the local lifecycle store.
- Audit path writable.
- Optional `AXIS_OPERATOR_TOKEN` configured only if you want lifecycle mutation protection.

The Docker Compose path starts both AXIS and a local PostgreSQL database:

```cmd
docker compose up --build
```

## API Demo Commands

These examples use Windows CMD-compatible `curl.exe` commands.

Health:

```cmd
curl.exe http://localhost:6543/health
```

Runtime stats:

```cmd
curl.exe http://localhost:6543/runtime/stats
```

Safe read query, expected `ALLOW`:

```cmd
curl.exe -X POST http://localhost:6543/query -H "Content-Type: application/json" -d "{\"actor\":\"reviewer\",\"app\":\"demo\",\"tenant\":\"acme\",\"role\":\"developer\",\"host\":\"localhost\",\"env\":\"prod\",\"sql\":\"SELECT 1 AS axis_demo\"}"
```

Dangerous `DELETE` without `WHERE`, expected `BLOCK` under the default policy:

```cmd
curl.exe -X POST http://localhost:6543/query -H "Content-Type: application/json" -d "{\"actor\":\"reviewer\",\"app\":\"demo\",\"tenant\":\"acme\",\"role\":\"developer\",\"host\":\"localhost\",\"env\":\"prod\",\"sql\":\"DELETE FROM users\"}"
```

Approval-required batch update under the default policy, expected `REQUIRE_APPROVAL`:

```cmd
curl.exe -X POST http://localhost:6543/query -H "Content-Type: application/json" -d "{\"actor\":\"reviewer\",\"app\":\"demo\",\"tenant\":\"acme\",\"role\":\"developer\",\"host\":\"localhost\",\"env\":\"prod\",\"sql\":\"UPDATE orders SET status = 'reviewed'\"}"
```

List approvals:

```cmd
curl.exe http://localhost:6543/approvals
```

Resolve an approval after copying an `approval_id` from the previous response:

```cmd
curl.exe -X POST http://localhost:6543/approvals/<approval_id>/resolve -H "Content-Type: application/json" -d "{\"actor\":\"dba-oncall\",\"decision\":\"approve\",\"comment\":\"Reviewed local demo approval\"}"
```

Recent audit events:

```cmd
curl.exe "http://localhost:6543/audit?limit=10"
```

Evidence verification:

```cmd
curl.exe http://localhost:6543/evidence/verify
```

Policy status:

```cmd
curl.exe http://localhost:6543/policy/status
```

Policy versions:

```cmd
curl.exe http://localhost:6543/policy/versions
```

Policy dry-run, expected no SQL execution and no audit write:

```cmd
curl.exe -X POST http://localhost:6543/policy/dry-run -H "Content-Type: application/json" -d "{\"mode\":\"current\",\"request\":{\"actor\":\"reviewer\",\"app\":\"demo\",\"tenant\":\"acme\",\"role\":\"developer\",\"host\":\"localhost\",\"env\":\"prod\",\"sql\":\"DELETE FROM users\"}}"
```

Policy validation using a minimal valid candidate. This candidate is useful for endpoint shape and warning review; do not activate it as a real pilot policy without adding the required protections.

```cmd
curl.exe -X POST http://localhost:6543/policy/validate -H "Content-Type: application/json" -d "{\"policy\":{\"schema_version\":\"1.0\",\"policy_version\":\"review-minimal-candidate\",\"defaults\":{\"read\":\"ALLOW\",\"write\":\"BLOCK\",\"ddl\":\"REQUIRE_APPROVAL\"},\"write_rules\":[]}}"
```

Policy diff using the same minimal valid candidate:

```cmd
curl.exe -X POST http://localhost:6543/policy/diff -H "Content-Type: application/json" -d "{\"policy\":{\"schema_version\":\"1.0\",\"policy_version\":\"review-minimal-candidate\",\"defaults\":{\"read\":\"ALLOW\",\"write\":\"BLOCK\",\"ddl\":\"REQUIRE_APPROVAL\"},\"write_rules\":[]}}"
```

## Policy Candidate, Activation, and Rollback

For a safe lifecycle demo, use the control plane:

1. Open `http://localhost:3000/policy`.
2. Open **Validate Candidate**.
3. Click **Load Active**.
4. Change only `policy_version`, for example to `prod_main@review-demo`.
5. Validate the candidate.
6. Open **Diff Candidate** and confirm the diff is low risk.
7. Open **Dry-Run** and test representative SQL.
8. Return to **Validate Candidate**, add an operator note, and create a candidate.
9. Open **Versions**, copy the candidate hash, and activate the candidate.
10. Confirm runtime status and policy status.
11. Roll back to the previous archived version by confirming its hash.

If an operator token is configured, set it only in the Control Plane server environment before starting Next.js. The browser UI must not receive or edit the token.

## Local Demo Script

The repository includes a small API-level script that uses only the Python standard library:

```cmd
python scripts/demo_axis_flow.py
```

Optional backend URL override:

```cmd
python scripts/demo_axis_flow.py --base-url http://localhost:6543
```

## Control Plane Demo Steps

1. Open the dashboard at `http://localhost:3000`.
2. Confirm backend health and runtime cards.
3. Open Query Console.
4. Send the safe `SELECT 1 AS axis_demo` query.
5. Send the risky `DELETE FROM users` query and confirm it is blocked.
6. Send the batch `UPDATE orders SET status = 'reviewed'` query and confirm approval is required.
7. Open Approvals and resolve the pending approval.
8. Open Evidence Explorer and inspect recent events.
9. Open Runtime and run evidence verification.
10. Open Policy, run validation, diff, dry-run, inspect versions, activate a safe candidate, and roll back.

## Expected Observations

- Safe read returns HTTP `200` with decision `ALLOW`.
- Dangerous `DELETE FROM users` returns HTTP `403` with decision `BLOCK`.
- Batch update returns HTTP `202` with decision `REQUIRE_APPROVAL` and an `approval_id`.
- Approval resolution records audit evidence and only executes when approved.
- Audit explorer shows real events, newest first.
- Evidence verification reports whether the hash chain is valid.
- Dry-run returns `would_execute: false` and `audit_written: false`.
- Policy versions show active, candidate, and archived records.

## Troubleshooting

No audit events:

- Run a query first.
- Check `AUDIT_LOG_PATH`.
- Check backend logs.
- Confirm the AXIS process can write the audit path.

Evidence invalid:

- Legacy or unhashed records may show invalid.
- Manually edited or malformed records should be investigated.
- AXIS reports corruption instead of repairing it silently.

Approval resolution fails:

- Confirm the `approval_id` is pending.
- Use `decision` values `approve` or `reject`.
- If the approval was already resolved, AXIS should reject a second resolution.

Policy lifecycle mutation returns `401`:

- `AXIS_OPERATOR_TOKEN` is configured.
- Send the `X-AXIS-Operator-Token` header or configure the token in control-plane Settings.
