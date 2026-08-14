# AXIS Demo Script

Run from the repository root on Windows PowerShell.

## 1. Start Docker Runtime

```powershell
docker compose down
docker compose up --build -d
docker compose ps
```

## 2. Check Health

```powershell
curl.exe http://localhost:6543/health
```

Expected: `status` is `ok`, policy integrity is `valid`, and audit WAL status is `healthy`.

## 3. Check Runtime Metrics

```powershell
curl.exe "http://localhost:6543/runtime/metrics"
```

Expected: top-level `status`, `requests`, `decisions`, `limits`, `policy`, and `audit`.

## 4. Send Safe SELECT

```powershell
$body = @{
  actor = "demo-reviewer"
  app = "axis-demo"
  tenant = "acme"
  role = "reader"
  host = "demo"
  env = "prod"
  sql = "SELECT * FROM orders ORDER BY id"
} | ConvertTo-Json

curl.exe -H "Content-Type: application/json" -d $body http://localhost:6543/query
```

Expected: HTTP 200 and `decision` is `ALLOW`.

## 5. Send Blocked Dangerous Query

```powershell
$body = @{
  actor = "demo-reviewer"
  app = "axis-demo"
  tenant = "acme"
  role = "admin"
  host = "demo"
  env = "prod"
  sql = "DELETE FROM orders"
} | ConvertTo-Json

curl.exe -H "Content-Type: application/json" -d $body http://localhost:6543/query
```

Expected: no `ALLOW`. The current policy blocks delete without `WHERE`.

## 6. Trigger Approval

```powershell
$body = @{
  actor = "demo-reviewer"
  app = "axis-demo"
  tenant = "acme"
  role = "admin"
  host = "demo"
  env = "prod"
  sql = "UPDATE orders SET status = 'demo_requires_approval'"
} | ConvertTo-Json

curl.exe -H "Content-Type: application/json" -d $body http://localhost:6543/query
```

Expected: HTTP 202 and `decision` is `REQUIRE_APPROVAL`.

## 7. Resolve Approval

List approvals:

```powershell
curl.exe http://localhost:6543/approvals
```

Reject an approval:

```powershell
$approvalId = "<approval_id from previous response>"
$body = @{
  actor = "demo-reviewer"
  decision = "reject"
  comment = "demo rejection"
} | ConvertTo-Json

curl.exe -H "Content-Type: application/json" -H "X-AXIS-Operator-Token: axis-local-dev-only-operator-token-000000000000" -d $body "http://localhost:6543/approvals/$approvalId/resolve"
```

Expected: rejected approval returns a block decision and does not execute.

## 8. Open Audit Events

```powershell
curl.exe "http://localhost:6543/audit/events?limit=5"
```

## 9. Verify Audit Chain

```powershell
curl.exe "http://localhost:6543/audit/verify"
```

Expected: `status` is `verified`.

## 10. Export Evidence Bundle V1

```powershell
curl.exe "http://localhost:6543/audit/export?limit=10" -o axis-evidence-bundle-v1.json
```

## 11. Verify Evidence Bundle

```powershell
python scripts/axis_evidence_bundle_verify.py --bundle axis-evidence-bundle-v1.json
```

Expected: `AXIS Evidence Bundle Verification: PASS` or `PASS_WITH_WARNINGS` for unsigned bundles.

## 12. Run Smoke Tests

```powershell
python scripts/axis_audit_api_smoke.py --base http://localhost:6543
python scripts/axis_runtime_smoke.py --base http://localhost:6543
```

## 13. Run Stress Test

The stress test is local-only and bounded.

```powershell
python scripts/axis_runtime_stress.py --base http://localhost:6543 --concurrency 25 --requests 500 --include-export --include-rate-limit
```

Expected: `AXIS Runtime Stress Validation: PASS`.

## 14. Stop Environment

Stop while preserving local evidence volumes:

```powershell
docker compose down
```

Reset local Docker volumes:

```powershell
docker compose down -v
```

Use reset only when local evidence can be discarded.
