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
# AXIS Pilot Checklist

## Pilot Goal

Evaluate whether AXIS can enforce deterministic policy on database write paths and provide verifiable evidence for risky operations.

## Pilot Evaluation Flow

```mermaid
flowchart TD
  A[Define pilot goal] --> B[Prepare non-production PostgreSQL]
  B --> C[Collect representative SQL operations]
  C --> D[Configure policy pack]
  D --> E[Run deterministic demo]
  E --> F[Inspect audit and evidence]
  F --> G[Test policy lifecycle]
  G --> H[Review gaps and rollout risks]
```

## Pre-Pilot Technical Requirements

- Test PostgreSQL instance.
- Non-production dataset.
- Representative SQL operations from applications, jobs, scripts, and migrations.
- Sample application or scripts that can send SQL through AXIS.
- Policy requirements for allowed writes, blocked writes, and approval-required writes.
- Operator contact for approval workflow testing.
- Test approval workflow participants.
- Log retention expectations.
- Agreement on acceptable latency overhead for the pilot.
- Agreement on what evidence must be reviewed after the pilot.

## Pilot Scope

- One database.
- Limited roles.
- Defined write operations.
- Defined policy pack.
- Defined approval users.
- Non-production or staging first.
- Explicit start and end criteria.

## Out of Scope for v0.6

- Broad enterprise deployment.
- Compliance attestation.
- High availability cluster.
- Multi-region policy sync.
- Full identity provider integration.
- Production-grade RBAC.
- External ledger or KMS-backed evidence.

## Success Criteria

- Risky writes are blocked or routed to approval according to policy.
- Reads are unaffected where intended.
- Audit events are generated for important decisions.
- Evidence chain verifies after normal operation.
- Policy dry-run predicts behavior for representative SQL.
- Rollback works for a stored valid policy version.
- Operators can understand decisions from the control plane.
- Reviewers can reproduce the demo without hidden setup.

## Failure Criteria

- Destructive write is allowed contrary to policy.
- Audit evidence is missing for an enforced decision.
- Evidence chain becomes unverifiable after normal operation.
- Policy activation leaves the system without a valid policy.
- Approval-required operation executes before approval.
- Approval bypass succeeds.
- Decision result is unexplained or not traceable to policy/classification.

## Review Questions

- What queries are in scope?
- What policy rules are required?
- Who approves risky writes?
- What evidence must be retained?
- What integrations are needed?
- What latency overhead is acceptable?
- Which operator actions require stronger identity controls?
- Which logs or evidence must leave the local host?
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
# Dirty Data Pilot Notes

Dirty data exists to avoid toy-data review. Dirty data mode is optional and must not slow down quickstart by default.

The generator in `scripts/generate_dirty_pilot_data.py` creates deterministic synthetic data for:

- users
- customers
- accounts
- transactions
- admin_actions

It includes nulls where the schema allows them, JSONB metadata columns, unicode and edge-case strings, old timestamps, soft-deleted records, multiple tenants, suspicious admin actions, and bulk-operation candidate records.

100k rows are not an enterprise-scale performance claim. This is realism for behavior and integration testing, not a final benchmark.

Benchmark claims require separate controlled benchmark work with defined hardware, dataset shape, workload, warmup, repetitions, and measurement methodology.

No real people, companies, or secrets should be generated.

Example:

Windows:

```powershell
python scripts\generate_dirty_pilot_data.py --rows 10000 --output demo-evidence-output\dirty-pilot-data.sql
```

macOS/Linux:

```bash
python3 scripts/generate_dirty_pilot_data.py --rows 10000 --output demo-evidence-output/dirty-pilot-data.sql
```

Load manually into the pilot database only when you want dirty-data behavior review:

```bash
docker compose -f demo/docker-compose.pilot.yml exec -T pilot-postgres psql -U varux -d prod_main < demo-evidence-output/dirty-pilot-data.sql
```
# AXIS Pilot Evidence Package

## Directory map

```text
evidence/
  pilot-v1/
    README.md
    clean-run/
      commands.md
      smoke-test-output.txt
      demo-run-output.txt
      compose-ps-output.txt
    health/
      axis-health.json
      backend-health.json
      frontend-health.txt
    responses/
      safe-read-response.json
      safe-write-response.json
      approval-required-response.json
      approval-retry-success-response.json
      approval-rejected-response.json
      blocked-operation-response.json
      transaction-safe-response.json
      transaction-approval-rollback-response.json
      transaction-blocked-rollback-response.json
    audit/
      audit-sample-events.json
      audit-verification-output.json
      audit-hash-chain-notes.md
    screenshots/
      README.md
    limitations/
      current-limitations.md
      native-wire-gap.md
      sql-alchemy-integration-notes.md
      approval-retry-model.md
      transaction-model.md
```

## How evidence is captured

`scripts/capture_pilot_evidence.py` calls the running pilot stack over HTTP. It does not generate fake policy decisions, fake approval ids, fake audit ids, or placeholder screenshots.

The script captures:

- AXIS health from `GET http://localhost:6654/health`.
- Backend health from `GET http://localhost:8000/health`.
- Frontend reachability from `GET http://localhost:8088`.
- Clean-run command outputs for smoke tests, demo run, and compose service status.
- Business responses for safe read, safe write, approval, rejection, blocked operation, and transaction workflows.
- AXIS audit events from `GET /audit/events?limit=100`.
- AXIS audit hash-chain verification from `GET /audit/verify` when available.

If capture stops before completion, it leaves `demo/evidence/pilot-v1/PARTIAL_CAPTURE.txt`. The verifier treats that as a failure.

## How to regenerate evidence

From the repository root:

```powershell
docker compose -f demo/docker-compose.pilot.yml down -v
docker compose -f demo/docker-compose.pilot.yml up -d --build
python scripts\pilot_smoke_tests.py
python scripts\run_pilot_demo.py
python scripts\capture_pilot_evidence.py
python scripts\verify_pilot_evidence.py
```

The capture script also archives smoke-test and demo output by invoking those same Python scripts during capture. This ensures the evidence directory contains reviewable command output even when a reviewer runs the commands interactively.

## How to verify evidence

Run:

```powershell
python scripts\verify_pilot_evidence.py
```

Expected success output:

```text
PILOT_EVIDENCE_VERIFICATION: PASS
```

On failure, the verifier prints each missing or invalid evidence item and then:

```text
PILOT_EVIDENCE_VERIFICATION: FAIL
```

## What each response file proves

- `safe-read-response.json`: the sample app can read business data directly from PostgreSQL and labels that path as `direct_postgres`.
- `safe-write-response.json`: a normal SQLAlchemy ORM insert is routed through AXIS, allowed by policy, executed, and audited.
- `approval-required-response.json`: a risky role update returns structured `approval_required` state with an AXIS `approval_id`.
- `approval-retry-success-response.json`: an approved operation succeeds only after the original request is explicitly retried with `approval_id`.
- `approval-rejected-response.json`: a rejected approval blocks the retry and returns structured `approval_rejected` state.
- `blocked-operation-response.json`: an unscoped destructive delete is blocked by AXIS policy and not executed.
- `transaction-safe-response.json`: a multi-write business workflow can route multiple safe writes through AXIS and complete.
- `transaction-approval-rollback-response.json`: a workflow that hits an approval-required operation rolls back local pending state; `marker_persisted` and the state check must both prove no partial marker.
- `transaction-blocked-rollback-response.json`: a workflow that hits a blocked destructive operation rolls back local pending state.

## How audit evidence should be inspected

Start with `audit-verification-output.json`. For the current pilot stack, `/audit/verify` should report `status: verified` and a positive `events_checked` count.

Then inspect `audit-sample-events.json`. Reviewers should look for:

- `event_id`
- `decision`
- `reason` or `reason_code`
- `risk`
- `matched_rule`
- `policy.policy_id`, `policy.policy_version`, and `policy.policy_sha256`
- `sql_fingerprint`
- `approval_id` for approval-related events
- `event_hash`
- `previous_hash`

`audit-hash-chain-notes.md` explains what AXIS verified automatically and what remains manual or future work.

## How to interpret PASS/FAIL results

`PASS` means the evidence directory is complete and internally consistent for the current pilot claim.

`FAIL` means a reviewer should not treat the package as complete. The printed failure lines identify exactly which file, field, command output, policy state, rollback state, or audit item needs attention.
# AXIS Pilot Limitations And Next Steps

## No native PostgreSQL wire proxy in this version

This pilot does not implement native PostgreSQL wire protocol compatibility. AXIS is reached through HTTP `/query` for protected writes. It is not demonstrated here as a transparent PostgreSQL listener that arbitrary PostgreSQL clients can use without application changes.

## HTTP adapter integration model

The FastAPI backend uses an AXIS HTTP adapter. The application constructs normal SQLAlchemy ORM operations, the custom session compiles protected writes into SQL, and the adapter submits them to AXIS HTTP `/query`.

This is an application integration model, not a driver-level proxy model.

## SQLAlchemy routing/interception is v1, not transparent universal ORM support

`AxisRoutingSession` is a pilot integration. It covers the ORM and SQLAlchemy Core patterns used by the sample business app. It is not a complete SQLAlchemy dialect and has not been proven against arbitrary flush ordering, relationship cascades, custom type binders, stored procedures, server-generated non-primary values, or every bulk operation mode.

## Read/write split is intentional

The pilot intentionally sends safe reads directly to PostgreSQL and protected writes through AXIS. This keeps the demo focused on policy-controlled mutation paths. It does not prove universal inspection of every read query.

## Human approval does not keep database transactions open

The pilot does not keep PostgreSQL transactions, locks, or connection state open while waiting for a human approval. Holding transactions open for human workflows would be operationally fragile and is not the model demonstrated here.

## Approval uses rollback + explicit retry model

When AXIS returns `REQUIRE_APPROVAL`, the application rolls back local SQLAlchemy transaction state. An operator approves or rejects through AXIS. If approved, the original business operation is submitted again with `approval_id`.

This proves approval-controlled execution. It does not prove suspended in-place database transaction continuation.

## Control Plane visibility depends on currently implemented endpoints

The optional Control Plane profile can show AXIS state where supported by existing AXIS endpoints. If a Control Plane view is incomplete, reviewers should inspect the backend endpoints and evidence files directly.

## Audit evidence depends on existing AXIS audit APIs

The evidence package uses AXIS `/audit/events` and `/audit/verify`. The package does not independently recompute every hash-chain value outside AXIS. Independent offline verification can be added later.

## This pilot is not a production deployment guide

The compose stack, local operator token defaults, sample policy, sample database, and frontend are for reviewer validation. They are not a hardened production deployment recipe.

## Next step: reviewer feedback

Use reviewer feedback to identify which evidence items need stronger proof, which integration boundaries need clarification, and which production questions must be answered before a customer pilot.

## Next step: native wire feasibility research

Evaluate a native PostgreSQL wire-compatible path, including protocol parsing, authentication, prepared statements, transaction/session affinity, connection pooling, and compatibility with common PostgreSQL drivers.

## Next step: second stack demo, likely Java/Spring Boot + Hibernate

Build a second integration using a different enterprise stack, likely Java, Spring Boot, and Hibernate. The goal is to test whether AXIS policy, approval, rollback, and audit concepts transfer beyond Python and SQLAlchemy.

## Next step: pilot customer environment mapping

Map the integration model to real customer environments: database topology, ORM usage, CI/CD controls, policy ownership, approval workflow, audit retention, operational monitoring, and rollback requirements.
# AXIS Pilot Benchmark Guide

## 1. Performance Philosophy: Measurement over Assumption

AXIS does not compete with raw database latency.
AXIS competes with uncontrolled production execution risk.

The pilot benchmark measures latency honestly by separating semantically different paths. It does not collapse safe reads/writes, approval-required work, and blocked dangerous SQL into one average latency number. AXIS may add measurable overhead. That overhead is evaluated against deterministic production write-path control, approval safety, auditability, and signed evidence generation.

## 2. Data Paths

Direct Path:

Application / Agent -> PostgreSQL

AXIS HTTP Path:

Application / Agent -> AXIS Execution Gateway (/query API) -> Policy Engine -> PostgreSQL

AXIS Approved Path:

Application / Agent (+approval_id) -> AXIS Execution Gateway -> CAS State Verification -> PostgreSQL

Native Wire Path:

Application / Agent -> AXIS Native Proxy -> PostgreSQL

The Control Plane is a management path, not the data path. Benchmark reports must not describe query execution as `Application -> AXIS Control Plane -> PostgreSQL`.

## 3. Benchmark Corpus Split

The default pilot benchmark uses a 70/20/10 corpus split:
70% clean allow-path queries, 20% approval-required controlled-risk queries, and 10% blocked dangerous or ambiguous queries.

The split is configurable with:

```powershell
python scripts/baseline_benchmark.py ^
  --queries 10000 ^
  --allow-ratio 70 ^
  --approval-ratio 20 ^
  --block-ratio 10 ^
  --output-format markdown,json
```

The benchmark validates that ratios sum to 100 and that `--queries` is greater than zero. A fixed `--seed` makes corpus selection deterministic.

## 4. Why Direct PostgreSQL Comparison Uses Only ALLOW Corpus

Direct PostgreSQL comparison is performed only against the allow-path corpus. Approval-required and blocked queries are measured as AXIS enforcement paths, because their purpose is to prevent unsafe execution rather than compete with raw database execution latency.

Dangerous SQL is never sent directly to PostgreSQL by the benchmark. DROP, TRUNCATE, blind DELETE, blind UPDATE, multi-statement write chains, unsupported SQL, malformed SQL, and unresolved prepared statements are sent only through AXIS, where they must fail closed before database execution.

## 5. Metrics Collected

Section 1 compares only Direct PostgreSQL ALLOW Path and AXIS HTTP `/query` ALLOW Path:

- p50, p95, and p99 latency
- min, max, and average latency
- throughput QPS
- error rate
- total, successful, and failed queries

Section 2 measures AXIS-only enforcement behavior for `REQUIRE_APPROVAL` and `BLOCK`:

- p50, p95, and p99 decision latency
- average decision latency
- approval record creation success count
- blocked count
- structured error count
- error rate

Section 3 summarizes security outcomes:

- total corpus size
- ALLOW, REQUIRE_APPROVAL, and BLOCK counts
- queries executed against DB
- queries not executed
- approval records created
- blocked before DB
- failed-closed count
- policy_id, policy_version, and policy_sha256 when available

## 6. Human Approval Time Exclusion

The approval-required measurement stops when AXIS returns `REQUIRE_APPROVAL`, creates the approval record, and reports `execution_status=NOT_EXECUTED`. Human wait time is not included.

Approved retry execution, when tested, is a separate path:

Application / Agent (+approval_id) -> AXIS Execution Gateway -> CAS State Verification -> PostgreSQL

That retry path must not be mixed into the ALLOW baseline comparison.

## 7. Native Wire Path Handling

Native Wire Path is optional. The benchmark checks it only when `--native-wire-check` is passed.

If a real native proxy is not enabled and reachable, the report marks:

```text
Native Wire Path: SKIP
Reason: native wire proxy is not enabled in this build.
```

The benchmark must not fake native wire behavior, SQLSTATE values, or support claims.

## 8. Report Interpretation

The Markdown report is written to:

```text
demo/benchmark-results/axis-benchmark-report.md
```

The JSON report is written to:

```text
demo/benchmark-results/axis-benchmark-report.json
```

Read the report in three sections. Section 1 answers the fair latency question for safe operations. Section 2 answers how quickly AXIS makes enforcement decisions. Section 3 answers what AXIS prevented, suspended, or allowed.

Do not average Section 1 and Section 2 together. A blocked DROP TABLE and a direct `SELECT 1` are not competing operations.

## 9. Objection Handling

Objection: "Why not compare blocked SQL to PostgreSQL?"

Answer: because direct execution of blocked SQL would damage or mutate the database. Timing a direct DROP, TRUNCATE, blind DELETE, or ambiguous multi-statement write is not benchmarking. It is self-sabotage with timing numbers.

Objection: "AXIS adds latency."

Answer: latency is measured directly in Section 1. The pilot question is whether that measured overhead is acceptable for deterministic production write-path control, approval safety, auditability, and verifiable evidence.

Objection: "Approval-required operations are slower than direct PostgreSQL."

Answer: approval-required operations are intentionally not direct database operations. Their purpose is to suspend controlled-risk work, create durable approval evidence, and prevent execution until an authorized retry supplies an approval id.

Objection: "Blocked operations have no PostgreSQL baseline."

Answer: correct. Their success condition is non-execution before the database, with structured evidence. Raw database latency is not the relevant comparison for a prevented unsafe operation.

## 10. Running the Benchmark

Default pilot run:

```powershell
python scripts/baseline_benchmark.py ^
  --queries 10000 ^
  --allow-ratio 70 ^
  --approval-ratio 20 ^
  --block-ratio 10 ^
  --output-format markdown,json
```

Common local run:

```powershell
python scripts/baseline_benchmark.py --queries 100 --output-format markdown,json
```

Useful options:

- `--axis-url http://localhost:6543`
- `--pg-host localhost`
- `--pg-port 5432`
- `--pg-user varux`
- `--pg-password varux`
- `--pg-database prod_main`
- `--tenant acme_corp`
- `--env production`
- `--actor benchmark_runner`
- `--actor-type service`
- `--output-dir ./demo/benchmark-results`
- `--concurrency 1`
- `--warmup 100`
- `--seed 42`
- `--native-wire-check`

If AXIS is not reachable, the benchmark reports an environment error and exits without treating that as benchmark failure:

```text
[ENVIRONMENT ERROR] AXIS is not reachable at <axis-url>.
This is not a benchmark failure. Start AXIS and rerun.
```

If PostgreSQL is not reachable, the benchmark reports:

```text
[ENVIRONMENT ERROR] PostgreSQL is not reachable at <host:port>.
This is not an AXIS failure. Check database service and credentials.
```

Final proof sentence:

Latency is measured.
Risk reduction is demonstrated.
Evidence is verifiable.
# AXIS Pilot Reviewer Demo Flow

## What this demo proves

AXIS can protect ORM-generated PostgreSQL write paths in a realistic business backend by routing protected operations through deterministic policy evaluation, approval handling, transaction-safe rollback behavior, and auditable evidence generation.

The pilot uses a FastAPI sample business app, SQLAlchemy ORM models, PostgreSQL, and an `AxisRoutingSession` adapter. Reads execute directly against PostgreSQL. Protected writes are compiled by SQLAlchemy and sent to AXIS HTTP `/query` for policy evaluation and execution.

## What this demo does not claim

This demo does not claim native PostgreSQL wire compatibility. It does not claim transparent driver-level interception for arbitrary applications. It is not a universal SQLAlchemy dialect, a production deployment guide, or proof that AXIS can be dropped in front of every enterprise database stack without application integration work.

The approval model does not hold database transactions open while a person approves. AXIS returns an approval requirement, the application rolls back local transaction state, an operator approves or rejects, and the original operation is explicitly retried with `approval_id`.

## Required services

- `pilot-postgres`: PostgreSQL sample database.
- `axis-backend`: AXIS backend with the pilot policy mounted.
- `sample-business-backend`: FastAPI business API using SQLAlchemy and `AxisRoutingSession`.
- `sample-business-frontend`: static admin UI at `http://localhost:8088`.
- Optional `control-plane`: AXIS Control Plane profile at `http://localhost:3000`.

## Clean start commands

```powershell
docker compose -f demo/docker-compose.pilot.yml down -v
docker compose -f demo/docker-compose.pilot.yml up -d --build
python scripts\pilot_smoke_tests.py
python scripts\run_pilot_demo.py
python scripts\capture_pilot_evidence.py
python scripts\verify_pilot_evidence.py
```

Expected verifier result:

```text
PILOT_EVIDENCE_VERIFICATION: PASS
```

## Step 1: Open frontend

Open `http://localhost:8088`.

The frontend is a sample admin UI for users, customers, accounts, approval actions, and transaction workflows. It calls the FastAPI backend and displays structured AXIS outcomes such as `allowed`, `approval_required`, `blocked`, and `approval_rejected`.

## Step 2: Verify backend and AXIS health

Open or call:

```powershell
Invoke-WebRequest -UseBasicParsing http://localhost:8000/health
Invoke-WebRequest -UseBasicParsing http://localhost:6654/health
```

The backend health response should identify the sample backend and AXIS base URL. The AXIS health response should report `status: ok`, active policy metadata, and audit health.

## Step 3: Safe read direct path

Use the frontend refresh action or call:

```powershell
Invoke-WebRequest -UseBasicParsing http://localhost:8000/api/customers
```

Expected result: a structured response with `status: allowed` and `path: direct_postgres`.

Reviewer point: safe reads are intentionally not sent through AXIS in this pilot. The read/write split is explicit application behavior.

## Step 4: Safe write through AXIS

Use the frontend `Create Customer` action or call the backend `POST /api/customers` endpoint.

Expected result: `status: allowed`, AXIS `decision: ALLOW`, policy metadata, SQL fingerprint, and `audit_event_id`.

Reviewer point: a normal ORM insert is compiled by SQLAlchemy, routed to AXIS HTTP `/query`, evaluated by policy, executed against PostgreSQL, and audited.

## Step 5: Risky operation requiring approval

Use the frontend `Change Role` action with a privileged role such as `security_admin`, or call `POST /api/users/{id}/role`.

Expected result: HTTP `202`, `status: approval_required`, AXIS `decision: REQUIRE_APPROVAL`, and an `approval_id`.

Reviewer point: the backend does not crash or convert the policy decision into a generic 500. It returns a structured business response that can be shown to an operator.

## Step 6: Approval from Control Plane or approval endpoint

Approve from the Control Plane if the optional profile is running, or use the sample backend approval proxy:

```powershell
Invoke-WebRequest -UseBasicParsing `
  -Method POST `
  -ContentType application/json `
  -Body '{"decision":"approve","comment":"Reviewer approval"}' `
  http://localhost:8000/api/approvals/<approval_id>/resolve
```

Expected result: AXIS records the approval and instructs the caller to retry the same request with `approval_id`.

## Step 7: Explicit retry after approval

Retry the original business operation with the same intended role and the returned `approval_id`.

Expected result: HTTP `200`, `status: allowed`, `decision: ALLOW`, and audit evidence for the approved execution.

Reviewer point: approval is not implicit execution. The pilot demonstrates an explicit retry model with durable AXIS approval state.

## Step 8: Dangerous operation blocked

Use the frontend `Dangerous Delete` action or call:

```powershell
Invoke-WebRequest -UseBasicParsing `
  -Method POST `
  -ContentType application/json `
  -Body '{}' `
  http://localhost:8000/api/dangerous/delete-all-customers
```

Expected result: HTTP `403`, `status: blocked`, AXIS `decision: BLOCK`, a matched block rule, SQL fingerprint, and audit event id.

Reviewer point: destructive unscoped deletes are denied by policy before execution.

## Step 9: Transaction rollback on approval-required operation

Use the frontend `Approval Transaction` action or call `POST /api/workflows/transaction-requires-approval`.

Expected result: `status: approval_required` and `marker_persisted: false`.

Reviewer point: when the transaction encounters an approval-required operation, the local SQLAlchemy transaction is rolled back. The demo proves this through both the response and the captured state check in `demo/evidence/pilot-v1/responses/transaction-approval-rollback-response.json`.

## Step 10: Transaction rollback on blocked operation

Use the frontend `Blocked Transaction` action or call `POST /api/workflows/transaction-blocked`.

Expected result: `status: blocked` and `marker_persisted: false`.

Reviewer point: when a destructive operation is blocked, local pending work in the same business workflow is rolled back and no partial admin action marker remains.

## Step 11: Inspect audit evidence

Inspect:

- `demo/evidence/pilot-v1/audit/audit-sample-events.json`
- `demo/evidence/pilot-v1/audit/audit-verification-output.json`
- `demo/evidence/pilot-v1/audit/audit-hash-chain-notes.md`

The sample should include event ids, decisions, reasons, risk levels, matched rules, policy metadata, SQL fingerprints, approval ids where applicable, `event_hash`, and `previous_hash` where available.

## Step 12: Review limitations

Read:

- `docs/demo/PILOT_LIMITATIONS_AND_NEXT_STEPS.md`
- `demo/evidence/pilot-v1/limitations/current-limitations.md`
- `demo/evidence/pilot-v1/limitations/native-wire-gap.md`
- `demo/evidence/pilot-v1/limitations/sql-alchemy-integration-notes.md`
- `demo/evidence/pilot-v1/limitations/approval-retry-model.md`
- `demo/evidence/pilot-v1/limitations/transaction-model.md`

The limitations are part of the evidence package. They define the boundary of what the pilot proves.

## Expected reviewer conclusion

A reviewer should conclude that AXIS currently works as an application-layer protection path for a realistic FastAPI and SQLAlchemy backend using PostgreSQL. The pilot demonstrates deterministic policy decisions, structured business responses, approval retry behavior, rollback behavior around policy stops, frontend operator visibility, and audit evidence.

The reviewer should not conclude that AXIS is currently a native PostgreSQL wire proxy or a transparent enterprise drop-in replacement for all database access paths.
# Demo Docs

This section is for pilot and demonstration material. It intentionally stays separate from technical AXIS runtime docs.

Start here:

- `REAL_APP_INTEGRATION_PILOT.md`
- `PILOT_REVIEWER_DEMO_FLOW.md`
- `PILOT_EVIDENCE_PACKAGE.md`
- `PILOT_LIMITATIONS_AND_NEXT_STEPS.md`
- `PILOT_READY_BENCHMARK_GUIDE.md`

# AXIS Real Application Integration & ORM Resilience Pilot v1

This pilot is a realistic sample business application that uses SQLAlchemy ORM patterns while routing protected write paths through AXIS HTTP `/query`.

It does **not** claim native PostgreSQL wire compatibility. It does **not** fake transparent driver-level interception. The integration model is an application-layer SQLAlchemy routing session that compiles ORM-generated SQL and submits protected writes to AXIS.

## Run

```bash
docker compose -f demo/docker-compose.pilot.yml up --build
```

Open:

- Sample business UI: `http://localhost:8088`
- Sample business backend: `http://localhost:8000/health`
- AXIS backend: `http://localhost:6654/health`

Optional Control Plane:

```bash
docker compose -f demo/docker-compose.pilot.yml --profile control-plane up --build
```

Then open `http://localhost:3000`.

## Automated Smoke Test

With the compose stack running:

```bash
python scripts/pilot_smoke_tests.py
```

Demo walkthrough:

```bash
python scripts/run_pilot_demo.py
```

## Architecture

```text
demo/sample-business-app/
  backend/
    FastAPI app
    SQLAlchemy ORM models
    AxisRoutingSession
    AXIS HTTP client adapter
  frontend/
    Lightweight admin UI
  db/
    PostgreSQL schema and seed data
    pilot-only AXIS policy
scripts/
  run_pilot_demo.py
  pilot_smoke_tests.py
demo/docker-compose.pilot.yml
```

Services:

- `pilot-postgres`: PostgreSQL with `users`, `customers`, `accounts`, `transactions`, `admin_actions`.
- `axis-backend`: existing AXIS backend, mounted with the pilot policy.
- `sample-business-backend`: FastAPI business API using SQLAlchemy.
- `sample-business-frontend`: static admin UI.
- `control-plane`: optional existing AXIS Control Plane profile.

## SQLAlchemy Integration

The backend uses `AxisRoutingSession` in `demo/sample-business-app/backend/app/axis_session.py`.

Read path:

- `select(...)`, `session.scalars(...)`, and ORM reads execute directly through the normal SQLAlchemy PostgreSQL engine.
- The pilot records direct-read metrics at `/api/integration/metrics`.

Protected write path:

- `session.add(...)` plus `session.commit()` compiles pending ORM `INSERT`, `UPDATE`, and `DELETE` operations using SQLAlchemy's PostgreSQL dialect.
- `session.execute(update(...))`, `session.execute(delete(...))`, and protected textual write/DDL statements are routed through AXIS.
- The compiled SQL is sent to `POST /query`.
- AXIS executes allowed writes against PostgreSQL and emits audit evidence.

Metadata sent to AXIS:

- `tenant` from `tenant_id`
- `actor`
- `actor_type`
- `role`
- `env`
- `session_id`
- SQL text
- safely representable SQLAlchemy parameters inside `params.sqlalchemy_parameters`
- `request_id`, `operation_source`, and `business_operation` inside `params`

AXIS currently generates its own canonical `request_id`; the pilot carries the business request id in `params` because `/query` does not currently expose a top-level request id input.

## Business Operations

The UI and backend cover:

- List users, customers, accounts.
- Create customer.
- Update customer profile.
- Change user role.
- Adjust account balance.
- Delete customer.
- Bulk update customer records.
- Dangerous destructive delete attempt.
- Multi-write transaction workflow.
- Approval-required operation and explicit retry after approval.

## AXIS Outcome Handling

The backend preserves structured AXIS outcomes:

- `ALLOW` -> HTTP `200` or `201`
- `REQUIRE_APPROVAL` -> HTTP `202`
- `BLOCK` -> HTTP `403`
- `approval_rejected` -> HTTP `409`
- audit evidence failure -> HTTP `500` with a specific evidence-failure message
- execution unknown -> HTTP `500` with an execution-unknown state

The frontend renders these messages:

- `Operation completed successfully.`
- `This operation requires security approval before execution.`
- `This operation was blocked by AXIS policy.`
- `The approval request was rejected.`
- `The operation could not be completed because audit evidence could not be safely committed.`

The UI does not display raw stack traces or convert AXIS policy decisions into crash pages.

## Approval Retry Model

Flow:

1. A risky operation, such as a user role change, is submitted from the business UI.
2. The backend compiles the ORM-generated SQL and sends it to AXIS `/query`.
3. AXIS returns `REQUIRE_APPROVAL` with an `approval_id`.
4. The backend rolls back the local SQLAlchemy transaction state.
5. The frontend displays `This operation requires security approval before execution.`
6. The frontend stores the original operation and displays the `approval_id`.
7. An operator approves from the AXIS Control Plane or from the sample app approval proxy.
8. The frontend retries the same business request with `approval_id`.
9. The backend recompiles and resubmits the same ORM operation to AXIS.
10. AXIS validates the approval proof, executes the SQL, and emits audit evidence.
11. The frontend shows final success.
12. If rejected, the frontend shows `The approval request was rejected.`

The backend does not keep PostgreSQL transactions or row locks open while approval is pending.

## Transaction Model

The pilot demonstrates three transaction scenarios from the business app perspective:

- Scenario A, all-safe transaction: account update, transaction insert, and admin action insert are routed through AXIS and complete successfully.
- Scenario B, approval-required transaction: a risky user update is evaluated first, AXIS creates an approval request, and the local pending admin action is rolled back.
- Scenario C, blocked destructive transaction: a dangerous delete is blocked by AXIS and the local pending admin action is rolled back.

Important limitation: AXIS HTTP `/query` currently accepts single SQL statements and uses backend-managed PostgreSQL connections. It is not a transaction coordinator and does not expose native connection pinning, `BEGIN`/`COMMIT` session affinity, or PostgreSQL wire protocol behavior. This v1 pilot avoids partial state in blocked/approval scenarios by ordering risky statements before any safe writes are forwarded. Full atomic multi-statement AXIS-mediated transactions require a future wire-compatible or transaction-aware integration.

## Control Plane Visibility

Pilot-generated events are visible through existing AXIS endpoints and the Control Plane where available:

- decision
- risk level
- matched rule
- reason
- SQL fingerprint
- policy id
- policy version
- policy SHA-256
- approval id
- audit event id

AXIS audit endpoints expose hash-chain evidence fields such as `event_hash` and `previous_hash` where supported by the current backend. If a specific Control Plane view is unavailable in an environment, use `GET /audit/events` on the AXIS backend or the sample backend proxy `/api/axis/audit/events`.

## Known Limitations

- No native PostgreSQL wire compatibility in this version.
- No transparent psycopg/driver-level integration.
- SQLAlchemy integration is a pragmatic custom `Session`, not a universal SQLAlchemy dialect.
- Literal SQL is compiled for AXIS execution because current AXIS `/query` executes SQL text and does not bind PostgreSQL protocol parameters.
- Complex ORM cascades, relationship flush ordering, server-generated non-primary values, and arbitrary SQLAlchemy bulk parameter sets are intentionally out of scope.
- Multi-statement atomicity through AXIS is limited as described in the transaction model.
- Backend retry state is held by the frontend for the demo flow; a production integration should persist retry intents durably.

## Next Steps

- Add a first-class AXIS decision-only preflight API for transaction planning.
- Add a transaction-aware AXIS execution API or native PostgreSQL wire compatibility.
- Add a SQLAlchemy dialect or connection proxy with broader statement coverage.
- Persist approval retry intents server-side with integrity checks.
- Extend Control Plane views for operation source and business operation metadata from `params`.
# AXIS Nedir?

AXIS, production veritabanı operasyonlarını çalışmadan önce politika ile değerlendiren ve her kritik karar için doğrulanabilir audit evidence üreten deterministik bir kontrol katmanıdır.

Mevcut implementasyonda AXIS, Rust tabanlı bir HTTP servis olarak çalışır. Uygulama, internal tool, script, AI agent veya servisler SQL isteklerini AXIS'in `/query` endpoint'ine gönderir. AXIS isteği doğrular, SQL'i PostgreSQL odaklı parser/classifier ile sınıflandırır, aktif policy'ye göre karar verir ve sadece izin verilen işlemleri PostgreSQL adapter üzerinden yürütür.

## Hangi problemi çözer?

Production veritabanlarında asıl risk çoğu zaman "hiç yetkisi olmayan biri bağlandı" değildir. Daha sık görülen risk, yetkili bir servis veya operatörün yanlış, geniş kapsamlı veya geri dönüşü zor bir SQL çalıştırmasıdır.

Örnek riskler:

- `WHERE` olmayan `DELETE`
- geniş kapsamlı `UPDATE`
- yanlış tenant veya tablo üzerinde write
- `DROP TABLE`, `TRUNCATE`, şema değiştiren DDL
- tek istekte birden fazla statement
- prepared statement ile niyetin saklanması
- unsupported SQL shape'in yanlışlıkla güvenli sanılması

AXIS bu riskleri execution öncesinde görünür ve kontrol edilebilir hale getirir.

## Authorized execution neden risklidir?

Sadece identity veya bağlantı yetkisi yeterli değildir. Yetkili bir backend servisi, migration job'u, admin script'i veya AI agent yanlış SQL üretebilir. Veritabanı açısından bağlantı geçerli olabilir; fakat çalışacak operasyon production verisi için riskli olabilir.

AXIS bu nedenle sadece "kim bağlanıyor?" sorusuna bakmaz. "Ne çalışmak üzere?", "hangi tabloda?", "hangi kapsamda?", "hangi ortamda?", "hangi policy altında?" sorularını da sorar.

## WRITE, DELETE ve DDL neden kritiktir?

Read-only sorgular genellikle geri dönüşsüz veri değişimi yapmaz. Buna rağmen `SELECT FOR UPDATE`, `SELECT INTO` veya write-capable CTE gibi read görünümlü riskler ayrıca değerlendirilmelidir.

WRITE, DELETE ve DDL ise production durumunu doğrudan değiştirir:

| Operasyon | Risk |
|---|---|
| `INSERT` | yanlış tenant, eksik scope, kritik tabloya kayıt |
| `UPDATE` | geniş kapsamlı veri değişikliği |
| `DELETE` | geri dönüşü zor veri kaybı |
| DDL | tablo, kolon, index veya şema yapısının değişmesi |
| Multi-statement | güvenli görünen istekle tehlikeli ikinci statement'ın birlikte gelmesi |

AXIS özellikle bu write path üzerinde deterministik karar üretir.

## AXIS neyi korur?

AXIS, kendisinden geçen protected database operation akışını korur.

Koruduğu alanlar:

- SQL'in execution öncesi sınıflandırılması
- policy tabanlı `ALLOW`, `BLOCK`, `REQUIRE_APPROVAL` kararı
- approval gerektiren işlemlerin ilk istekte çalıştırılmaması
- onaylı retry için aynı request bağlamı, SQL fingerprint ve policy metadata kontrolü
- audit WAL üzerinde hash-chain evidence üretimi
- policy manifest ve SHA-256 doğrulaması
- fail-safe davranışla tehlikeli belirsizliklerin güvenli tarafa çekilmesi

## AXIS neyi korumaz?

AXIS bütün güvenlik problemlerini tek başına çözmez.

AXIS şunların yerine geçmez:

- IAM/RBAC
- network security
- database role separation
- backup ve restore planı
- monitoring ve alerting
- secrets management
- least privilege
- TLS/mTLS veya servis kimliği
- production deployment discipline
- harici tamper-proof ledger
- remote attestation

AXIS'ten tamamen bypass edilerek doğrudan PostgreSQL'e giden write trafiğini AXIS kontrol edemez. Production ortamda network, credential ve role separation ile protected write path'in AXIS'ten geçmesi zorunlu kılınmalıdır.

## AXIS ne değildir?

AXIS:

- sadece loglama aracı değildir
- sadece reverse proxy değildir
- WAF değildir
- IAM/RBAC yerine geçen tek başına bir sistem değildir
- AI agent değildir
- bütün güvenlik problemlerini otomatik çözen bir ürün değildir
- compliance certification kanıtı değildir
- native PostgreSQL wire protocol desteğini mevcut HTTP adapter modeliyle otomatik kanıtlamaz

AXIS'in değeri, production veritabanı operasyonu çalışmadan önce deterministik karar üretmesi ve kritik kararların audit evidence ile doğrulanabilir olmasındadır.

# Zihinsel Model

AXIS'i production veritabanının önündeki kontrollü kapı gibi düşünün. Bu kapı, gelen isteğin kimden geldiğine bakar; fakat asıl olarak çalıştırılmak üzere olan SQL'in ne yaptığını inceler.

Bu benzetme basittir, ama çocukça değildir: production verisi üzerinde işlem yapmadan önce kontrol noktası oluşturmak ciddi bir operasyonel güvenlik ihtiyacıdır.

## Model 1: Veritabanı kapısı

Uygulama doğrudan production PostgreSQL'e write göndermek yerine AXIS'e gelir. AXIS, isteği geçirebilir, durdurabilir veya onay isteyebilir.

```text
Uygulama / Script / Servis / AI Agent
                |
                v
        +----------------+
        | AXIS Kapısı    |
        | parse + policy |
        +----------------+
          |      |      |
          |      |      +--> Onay bekle
          |      +---------> Blokla
          +----------------> PostgreSQL'e gönder
```

## Model 2: Güvenlik kontrol noktası

AXIS bir firewall gibi paket bakmaz; SQL operasyonunun anlamına bakar. Örneğin `DELETE FROM users` ve `DELETE FROM users WHERE id = ? AND tenant_id = ?` aynı riskte değildir.

Kontrol edilen sinyaller:

- operasyon tipi: read, insert, update, delete, DDL
- scope: single, batch, unknown
- target: database, schema, table
- risk sinyalleri: `delete_without_where`, `bulk_operation`, `unknown_target`
- ortam: prod veya non-prod
- aktör tipi: human, service, ci_cd, ai_agent

## Model 3: Policy hakemi

Policy engine hakem gibi çalışır. İstek için bir karar üretir:

| Karar | Anlam |
|---|---|
| `ALLOW` | İstek çalıştırılabilir. |
| `BLOCK` | İstek PostgreSQL'e gönderilmez. |
| `APPROVAL_REQUIRED` / `REQUIRE_APPROVAL` | İstek ilk aşamada çalıştırılmaz; approval kaydı oluşur. |

Mevcut API'de karar adı `REQUIRE_APPROVAL` olarak görülebilir. Bu dokümanda `APPROVAL_REQUIRED`, aynı kavramı anlatmak için kullanılır.

## Model 4: Evidence kaydedici

AXIS sadece "izin verdim" veya "blokladım" demez. Kararın neden verildiğini audit evidence içine yazar.

Evidence şunları bağlar:

- request identity ve context
- SQL fingerprint
- classifier sonucu
- policy id, version ve SHA-256
- decision ve reason code
- approval id varsa approval bağı
- `previous_hash`
- `event_hash`

Bu sayede reviewer "ne oldu?" sorusuna normal runtime log yerine doğrulanabilir audit event üzerinden bakabilir.

## Kısa özet

AXIS'in zihinsel modeli dört parçadır:

```text
Kontrollü kapı
    -> SQL'i anlamaya çalışan sınıflandırıcı
    -> policy hakemi
    -> audit evidence kaydedici
```

Bu model, deployment disiplinine bağlıdır. AXIS'ten geçmeyen doğrudan database write path, AXIS tarafından korunmuş sayılmaz.

# Policy Before Execution

Policy-before-execution, AXIS'in temel güvenlik prensibidir. Türkçe karşılığıyla: işlem production'a ulaşmadan önce policy kararı verilmelidir.

Sadece "kim bağlanabilir?" sorusu yeterli değildir. Yetkili bir servis bile yanlış SQL üretebilir. Yetkili bir operatör de yanlış tenant, yanlış tablo veya geniş kapsamlı `DELETE` çalıştırabilir. AXIS bu nedenle "ne çalışmak üzere?" sorusunu execution öncesinde sorar.

## Geleneksel model

Geleneksel model çoğu zaman şu sırayla işler:

```text
identity -> access -> execution -> logs
```

Bu modelde loglar genellikle işlemden sonra oluşur. Eğer yanlış `DELETE` production'a ulaştıysa, log sadece ne olduğunu anlatır; zararı önlemez.

## AXIS modeli

AXIS modeli şu sırayı hedefler:

```text
identity/context -> operation inspection -> policy decision -> controlled execution -> evidence
```

Burada kritik fark, execution'ın policy kararından sonra gerçekleşmesidir.

## Ana akış

```mermaid
flowchart LR
  A[İstek] --> B[Parse ve sınıflandırma]
  B --> C[Policy değerlendirme]
  C --> D{Karar}
  D -->|ALLOW| E[Kontrollü execution]
  D -->|BLOCK| F[Blokla]
  D -->|APPROVAL_REQUIRED| G[Onay bekle]
  E --> H[Audit evidence]
  F --> H
  G --> H
```

## Neden execution öncesi?

Execution öncesi kontrol üç sebeple önemlidir:

1. Veri değişmeden önce risk yakalanır.
2. Approval gereken operasyonlar ilk istekte çalışmaz.
3. Karar, policy metadata ve SQL fingerprint ile birlikte kanıtlanabilir.

## Mevcut implementasyondaki karşılığı

Mevcut backend'de `/query` akışı kabaca şöyledir:

1. JSON body ve request alanları doğrulanır.
2. İsteğe bağlı JWT trusted context uygulanır.
3. SQL boyutu ve rate limit kontrol edilir.
4. Prepared statement komutları ayrıca ele alınır.
5. SQL parser/classifier tek statement analiz eder.
6. `PolicyEvaluator` aktif policy ile karar üretir.
7. `Enforcer` audit decision evidence yazmadan protected write execution'a ilerlemez.
8. Karara göre PostgreSQL execution, block veya approval record oluşturulur.

## Önemli ayrım

Runtime logs operasyonel görünürlük sağlar; audit evidence ise security proof olarak düşünülür. Policy-before-execution modeli, sadece karar üretmekle kalmaz; kararın neye göre üretildiğini audit WAL içine taşır.

# Mimari Harita

AXIS mevcut repo yapısında Rust backend, PostgreSQL adapter, policy lifecycle, SQLite approval store, WAL tabanlı audit evidence ve Next.js Control Plane bileşenlerinden oluşur.

Ana runtime path HTTP `/query` üzerindedir. Bu, native PostgreSQL wire protocol desteği anlamına gelmez; mevcut koruma modeli HTTP adapter veya uygulama entegrasyonu ile AXIS'e yönlendirilen protected operation akışları için geçerlidir.

## Genel mimari

```mermaid
flowchart LR
  Caller[Caller / Uygulama / Internal Tool / AI Agent] --> Listener[AXIS HTTP listener]
  Listener --> Parser[SQL parser ve classifier]
  Parser --> Policy[Policy engine]
  Policy --> Decision{Karar}
  Decision -->|ALLOW| Adapter[Database adapter]
  Adapter --> Pg[(Production PostgreSQL)]
  Decision -->|BLOCK| Block[Execution yok]
  Decision -->|APPROVAL_REQUIRED| Approval[Approval system]
  Approval --> Store[(Approval SQLite store)]
  Listener --> Audit[(Audit WAL)]
  Audit --> Projection[JSONL projection]
  Audit --> Verify[Evidence verification]
  Control[Control Plane UI] --> Proxy[Server-side proxy]
  Proxy --> Listener
```

## Bileşenler

| Bileşen | Sorumluluk | Mevcut repo karşılığı |
|---|---|---|
| Caller | SQL isteği gönderen uygulama, tool, script veya agent | Entegrasyonlara bağlı |
| AXIS HTTP listener | `/query`, approvals, audit, policy ve runtime endpoint'leri | `src/main.rs`, `src/gate/listener.rs` |
| SQL parser/classifier | Tek statement parse, operation/scope/target/risk çıkarımı | `src/gate/classifier.rs` |
| Policy engine | Aktif policy ile deterministik karar üretimi | `src/policy/evaluator.rs` |
| Approval system | Pending approval oluşturma, approve/reject, retry proof | `src/approval/store.rs`, listener approval akışı |
| Audit WAL | Durable source of truth evidence | `src/audit/logger.rs` |
| JSONL projection | WAL sonrası operator kolaylığı için projection | `AuditLogger::with_projection` |
| Database adapter | İzin verilen SQL'i PostgreSQL'e gönderme | `src/db/postgres.rs` |
| Production PostgreSQL | Protected target veritabanı | Deployment'a bağlı |
| Control Plane UI | Operatör görünürlüğü ve proxy tabanlı yönetim | `control-plane/` |
| Policy lifecycle | manifest, SHA-256, dry-run, activation, rollback | `src/policy/manifest.rs`, `src/policy/lifecycle.rs`, `src/policy/store.rs` |

## Mimari prensipler

- Deterministik karar: AI tahmini değil, parser/classifier ve policy sonucu.
- Fail-safe: belirsiz veya desteklenmeyen şekiller güvenli tarafa çekilir.
- Evidence-first protected write: protected write öncesi audit evidence yazılamazsa execution ilerlememelidir.
- WAL canonical source: audit WAL kanıt kaynağıdır; projection ve runtime logs yardımcıdır.
- Control Plane proxy sınırı: browser doğrudan backend URL veya operator token görmemelidir.

## Uygulandı ve limit

Uygulandı:

- HTTP `/query` gate
- PostgreSQL odaklı SQL sınıflandırma
- policy manifest SHA-256 doğrulaması
- approval store ve immutable resolve davranışı
- WAL hash-chain verification
- Control Plane server-side proxy

Limit:

- Native PostgreSQL wire protocol kapsamı ayrı değerlendirilmelidir.
- AXIS'ten bypass edilen doğrudan DB write trafiği AXIS tarafından korunmaz.
- Local manifest SHA-256 remote attestation değildir.

# Karar Akışı

Bir AXIS isteği önce request validation, sonra SQL classification, sonra policy evaluation adımlarından geçer. Karar verilmeden protected execution başlamaz.

## Karar akışı

```mermaid
flowchart TD
  A[Request alındı] --> B{Request geçerli mi?}
  B -->|Hayır| R[REJECT / BLOCK]
  B -->|Evet| C{SQL parse edilebilir mi?}
  C -->|Hayır| F[Fail-safe BLOCK]
  C -->|Evet| D[Operation, target, scope, risk çıkar]
  D --> E[Policy evaluate]
  E --> G{Decision}
  G -->|ALLOW| H[Audit decision yaz]
  H --> I[PostgreSQL execution]
  I --> J[Result evidence yaz]
  G -->|BLOCK| K[Audit block yaz]
  K --> L[Execution yok]
  G -->|APPROVAL_REQUIRED| M[Approval oluştur]
  M --> N[Audit approval yaz]
  N --> O[Execution yok, onay bekle]
```

## Karar modeli

| Karar | Anlam | Mevcut implementasyon notu |
|---|---|---|
| `ALLOW` | SQL execution'a gidebilir. | Protected write için decision evidence önce yazılır. |
| `BLOCK` | SQL PostgreSQL'e gönderilmez. | Policy, parser veya fail-safe nedeniyle olabilir. |
| `APPROVAL_REQUIRED` | İlk istek çalışmaz; approval kaydı oluşur. | API'de `REQUIRE_APPROVAL` olarak görünür. |
| `SUSPEND` | Kavramsal askıya alma halidir. | Ayrı policy enum'u değildir; approval bekleme, execution unknown veya manual review durumlarını anlatır. |
| `REJECT` | Approval çözümünde operatör reddidir. | Approval decision `reject`; final query decision çoğunlukla `BLOCK`, approval status `REJECTED`. |

## Örnekler

| SQL örneği | Beklenen sınıflandırma | Tipik karar | Neden |
|---|---|---|---|
| `SELECT 1` | Read | `ALLOW` | Safe read default allow olabilir. |
| `UPDATE orders SET status='x' WHERE id=1 AND tenant_id='acme'` | Scoped write | `ALLOW` veya `APPROVAL_REQUIRED` | Policy'ye bağlıdır; scoped write özel allow kuralı varsa geçebilir. |
| `UPDATE orders SET status='x'` | Batch write | `APPROVAL_REQUIRED` veya `BLOCK` | Scope geniş veya belirsizdir. |
| `DELETE FROM users` | Delete without WHERE | `BLOCK` | Kritik tablo ve geniş delete riski. |
| `DROP TABLE users` | DDL | `BLOCK` veya `APPROVAL_REQUIRED` | DDL production için kritik risk taşır. |
| `SELECT 1; DELETE FROM users` | Multi-statement | `BLOCK` | Tek istekte birden fazla statement reddedilir. |
| `PREPARE danger AS DELETE FROM users` | Prepared intent | policy'ye göre block/approval | AXIS iç SQL'i sınıflandırır, database-side PREPARE olarak kör geçirmez. |
| `EXECUTE danger` | Prepared execute | resolved ise orijinal SQL'e göre karar | Unresolved veya cross-session execute fail-safe block olur. |
| `VACUUM` veya desteklenmeyen şekil | Unsupported | `BLOCK` | Güvenli sınıflandırılamayan SQL fail-closed davranır. |

## Fail-safe davranış

Parser failure, unsupported SQL, ambiguous classification, missing context veya unresolved prepared statement gibi durumlar güvenli tarafa çekilir. Bu tür kararlar sadece runtime error gibi ele alınmaz; mümkün olduğunda audit evidence ile kaydedilir.

# Onay Akışı

Approval, riskli ama tamamen yasaklanması gerekmeyen operasyonları kontrollü hale getirmek için vardır. AXIS bu durumda ilk isteği çalıştırmaz; pending approval kaydı oluşturur ve audit evidence üretir.

## Approval neden var?

Bazı production işlemleri doğası gereği risklidir ama operasyonel olarak gerekli olabilir:

- geniş kapsamlı ama planlı `UPDATE`
- migration cleanup
- kontrollü DDL
- kritik tablodaki scoped ama hassas write
- AI agent veya servis tarafından üretilen write

Bu işlemleri doğrudan `ALLOW` yapmak risklidir. Her zaman `BLOCK` yapmak da operasyonu imkansız kılabilir. Approval bu iki durum arasında kontrollü bir durak sağlar.

## Yaşam döngüsü

```text
PENDING -> APPROVED -> EXECUTING -> EXECUTED
PENDING -> REJECTED
PENDING -> EXPIRED
EXECUTING -> EXECUTION_FAILED
EXECUTING -> REQUIRE_MANUAL_REVIEW
```

Mevcut store SQLite tabanlıdır. Record; request context, SQL fingerprint, classification, risk level, reason code, matched rule, policy id/version/SHA-256 ve audit event id bağlarını tutar.

## Sequence diagram

```mermaid
sequenceDiagram
  participant C as Caller
  participant A as AXIS
  participant P as Policy engine
  participant S as Approval store
  participant W as Audit WAL
  participant D as PostgreSQL

  C->>A: Riskli SQL isteği
  A->>P: Policy evaluate
  P-->>A: APPROVAL_REQUIRED
  A->>W: Approval created evidence
  A->>S: PENDING approval kaydı
  A-->>C: 202 ve approval_id
  C->>A: Operator approve/reject
  A->>W: Approval resolved evidence
  A->>S: Final state yaz
  alt Approved
    C->>A: Aynı request + approval_id ile retry
    A->>S: Exact-context reservation
    A->>W: Execution attempt evidence
    A->>D: SQL execution
    A->>W: Execution result evidence
    A-->>C: Sonuç
  else Rejected
    A-->>C: BLOCK / rejected
  end
```

## Pending approval nasıl oluşur?

Policy `REQUIRE_APPROVAL` kararı döndürdüğünde AXIS:

1. approval id üretir,
2. expiry zamanı belirler,
3. approval creation evidence yazar,
4. SQLite approval store içine `PENDING` kayıt ekler,
5. response içinde `approval_id` ve `NOT_EXECUTED` döndürür.

Bu aşamada SQL PostgreSQL'e gönderilmez.

## Approve/reject nasıl çözülür?

`POST /approvals/:approval_id/resolve` endpoint'i approval'ı çözer. Mevcut implementation operatör token gerektirir; token yoksa veya yanlışsa güvenli hata döner.

- `approve`: kayıt `APPROVED` olur; execution hemen burada yapılmaz. Caller aynı request context ve `approval_id` ile retry yapmalıdır.
- `reject`: kayıt `REJECTED` olur; original request blocked kalır.

## Final karar neden immutable olmalıdır?

Approval resolve sonrası aynı approval tekrar çözülemez. Bunun nedeni evidence zincirinin ve operasyonel anlamın bozulmamasıdır. Bir approval önce reject sonra approve yapılabiliyorsa audit trail güvenilirliğini kaybeder.

Mevcut store `PENDING` dışındaki kayıtların tekrar resolve edilmesini reddeder. Approved retry reservation da tek kullanımlıdır; aynı approval birden fazla execution için kullanılamaz.

## Reject durumunda neden evidence gerekir?

Reject de güvenlik kararıdır. Reviewer sadece "çalışmadı" demekle yetinmemelidir. Neden çalışmadığı, kim tarafından reddedildiği, hangi original policy kararına bağlı olduğu ve hangi approval id ile ilişkili olduğu audit evidence içinde görünmelidir.

## Request, policy ve evidence bağı

Approval record şu bağları korur:

- `approval_id`
- `request_id`
- SQL fingerprint
- classification
- original reason code
- matched rule
- `policy_id`
- `policy_version`
- `policy_sha256`
- created/resolved audit event id

Bu bağlar olmadan onay kararı ayrı bir iş akışı notu olarak kalır. AXIS'te approval, request ve audit evidence aynı güvenlik modelinin parçalarıdır.

# Audit Evidence

AXIS'te audit evidence, normal logdan daha güçlü bir kavramdır. Normal log operasyonel gözlem sağlar; audit evidence ise kararın, bağlamın ve zincir bütünlüğünün doğrulanmasını hedefler.

## Normal log neden yetmez?

Runtime loglar faydalıdır ama sınırlıdır:

- bellekte tutulabilir ve restart sonrası kaybolabilir,
- kapasite dolunca eski kayıtlar düşebilir,
- hash-chain kanıtı taşımaz,
- security proof olarak source of truth değildir.

AXIS runtime logs bu yüzden "operasyonel görünürlük" olarak ele alınır. Güvenlik kanıtı için audit WAL kullanılmalıdır.

## Audit evidence ne demektir?

Audit evidence, kritik decision veya execution olayı için şu bilgilerin kaydedilmesidir:

- request id
- actor, app, tenant, role, host, env
- SQL fingerprint ve normalize edilmiş SQL
- operation, query type, target, scope
- risk signals
- policy decision ve final decision
- reason code ve matched rule
- `policy_id`, `policy_version`, `policy_sha256`
- approval id varsa approval bağı
- execution status
- `previous_hash`
- `event_hash`

## WAL neden source of truth?

WAL, write-ahead log anlamında kullanılır. AXIS audit WAL'ı append-only JSON line kayıtlarından oluşur ve evidence için canonical source kabul edilir.

Mevcut backend'de audit visibility WAL'dan okur. JSONL projection ise WAL commit sonrası yazılan yardımcı bir projection'dır. Projection başarısız olursa WAL başarısı iptal edilmez; projection health degraded olabilir.

## JSONL projection ne işe yarar?

JSONL projection, operatör görünürlüğü ve pratik inceleme için yardımcıdır. Ancak security proof olarak WAL'ın yerini almaz.

| Katman | Rol |
|---|---|
| Audit WAL | Evidence source of truth |
| JSONL projection | Operator convenience |
| Runtime logs | Kısa ömürlü operasyonel özet |
| Audit derived index | Arama ve sayfalama için rebuild edilebilir index |

## `event_hash` nedir?

`event_hash`, audit event içeriğinin canonical formu üzerinden hesaplanan SHA-256 hash değeridir. Event'in gövdesi değişirse recompute edilen hash, kaydedilmiş `event_hash` ile eşleşmez.

## `previous_hash` nedir?

`previous_hash`, bir önceki audit event'in `event_hash` değeridir. İlk event için genesis convention olarak null olabilir.

Bu alan event'leri zincir haline getirir:

```text
Event 1: previous_hash = null
Event 1: event_hash = H1

Event 2: previous_hash = H1
Event 2: event_hash = H2

Event 3: previous_hash = H2
Event 3: event_hash = H3
```

## Hash-chain continuity

Hash-chain continuity, her event'in önceki event hash'ini doğru göstermesidir. Bir kayıt silinir, taşınır veya değiştirilirse zincir kopar.

```mermaid
flowchart LR
  E1[Event 1] -->|event_hash H1| E2[Event 2 previous_hash H1]
  E2 -->|event_hash H2| E3[Event 3 previous_hash H2]
  E3 --> V[Verify]
  V --> R[Zincir sonucu]
```

## Restart continuity neden önemli?

AXIS restart olduğunda WAL'ın son geçerli event hash'ini okur ve yeni event için `previous_hash` olarak onu kullanır. Bu davranış olmazsa restart sonrası evidence zinciri kopar veya yeni genesis başlatılmış gibi görünür.

Mevcut `AuditLogger` startup sırasında WAL'ı okuyup son hash'i yükler. Corrupt WAL startup'ta fail-fast davranışa yol açar.

## Tamper detection nasıl düşünülür?

Tamper detection, şu durumları yakalamayı hedefler:

- event gövdesi değişti,
- `event_hash` eksik,
- `previous_hash` eksik,
- hash zinciri yanlış sıraya geldi,
- JSON record bozuk,
- canonicalization hatası oluştu.

Bu model local dosya bütünlüğü sağlar; harici append-only ledger veya remote attestation iddiası değildir.

## Audit verification neyi kanıtlar?

`/evidence/verify` ve `/audit/verify` benzeri endpoint'ler hash-chain continuity ve event hash doğruluğunu kontrol eder. Başarılı verification şunu söyler:

- okunan WAL kayıtları kendi içinde hash olarak tutarlı,
- event gövdeleri kaydedilmiş hash ile uyumlu,
- zincir kopmamış.

Şunu kanıtlamaz:

- actor'ın iş niyeti doğruydu,
- external ticket doğruydu,
- database state business açısından doğru,
- local host tamamen güvenilirdi,
- remote attestation yapıldı.

## Evidence export ne işe yarar?

Evidence export, seçilmiş audit event'leri ve verification metadata'yı taşınabilir bir bundle içine koyar. Repo'da Evidence Bundle V1 ve offline verification script'leri bulunur. Evidence signing için Ed25519 destekleyen kod da vardır; signing etkin değilse bundle bunu `disabled` veya `none` olarak dürüstçe raporlamalıdır.

Bu, compliance certification değildir. Reviewer'a inceleme ve doğrulama materyali sağlar.

## Policy metadata neden evidence içine girer?

Karar tek başına yeterli değildir. Reviewer "hangi policy ile bu karar verildi?" sorusunu sorabilmelidir.

Bu yüzden evidence içinde şu alanlar önemlidir:

| Alan | Anlam |
|---|---|
| `policy_id` | Policy kimliği |
| `policy_version` | Policy versiyonu |
| `policy_sha256` | Active policy dosyasının raw SHA-256 değeri |

Policy değişiklikleri write-path davranışını değiştirdiği için karar anındaki policy metadata evidence içinde taşınmalıdır.

## Basitleştirilmiş audit event örneği

```json
{
  "event_type": "decision_made",
  "request_id": "req-123",
  "actor": "service-a",
  "app": "orders-api",
  "tenant": "acme",
  "env": "prod",
  "sql_fingerprint": "abc123...",
  "decision": "BLOCK",
  "policy_decision": "BLOCK",
  "reason_code": "bulk_delete_without_where",
  "matched_rule": "prod.bulk_delete_guard",
  "policy_id": "axis-prod-main",
  "policy_version": "prod_main@2026-05-01.1",
  "policy_sha256": "64-hex-karakter...",
  "previous_hash": "onceki-event-hash",
  "event_hash": "bu-event-hash"
}
```

Gerçek event yapısı daha fazla alan içerir ve raw SQL değerlerini redaction kurallarına göre sınırlar.

# Fail-Safe Davranış

AXIS'in güvenlik modeli riskli belirsizliklerde fail-closed davranmayı hedefler. Yani sistem bir işlemi güvenli şekilde değerlendiremiyorsa, production write path'i açık bırakmamalıdır.

## Fail-safe ve fail-closed

Fail-safe genel prensiptir: hata durumunda güvenli tarafa geç.

Fail-closed ise bu prensibin write path karşılığıdır: belirsizlik veya güvenlik kritik hata varsa execution kapalı kalır.

## Başlangıç fail-fast

AXIS başlangıçta bazı koşulları doğrular:

- audit WAL açılabilir ve zincir devam ettirilebilir olmalı,
- policy manifest okunabilir olmalı,
- active policy path policy dizini dışına çıkmamalı,
- SHA-256 manifest ile eşleşmeli,
- policy schema valid olmalı,
- activation dry-run corpus başarısız olmamalı,
- approval SQLite store bozuk olmamalı,
- production runtime profile zayıf operator token ile başlamamalı.

Bu kontroller başarısızsa sistemin trafik kabul etmemesi beklenir.

## Runtime protected execution safeguards

Protected write için önemli kural şudur:

```text
Durable audit decision evidence yazılamıyorsa protected execution ilerlememeli.
```

Mevcut enforcer, protected write öncesi decision evidence commit başarısız olursa DB call yapmadan block döndürür.

## Audit write failure behavior

Audit WAL yazılamıyorsa:

- protected write execution öncesi durdurulmalıdır,
- block veya service unavailable gibi güvenli response dönmelidir,
- runtime log güvenli özet yazabilir ama audit evidence yerine geçmez.

Execution gerçekleştikten sonra result evidence commit başarısız olursa durum farklıdır: PostgreSQL execution confirmed olabilir, fakat durable result evidence yoktur. Mevcut implementasyon bunu kritik integrity state olarak raporlar; sessiz başarı gibi göstermemelidir.

## Policy invalid behavior

Policy manifest eksik, checksum mismatch veya policy validation hatalıysa:

- başlangıç fail-fast olmalıdır,
- permissive default policy'ye sessiz fallback yapılmamalıdır,
- activation failure evidence yazılabiliyorsa yazılmalıdır.

## Approval corruption behavior

Approval store bozuksa unsafe resolve yapılmamalıdır. Mevcut store SQLite integrity check ve schema doğrulaması yapar; corrupt DB startup veya store açılışında hata üretir.

## Hata davranışı tablosu

| Durum | Beklenen davranış | Neden |
|---|---|---|
| Policy yüklenemiyor | Startup fail-fast | Yanlış policy ile write path açılmamalı |
| Policy checksum mismatch | Startup fail-fast | Local policy bütünlüğü bozulmuş olabilir |
| Audit WAL corrupt | Startup fail-fast | Evidence zinciri güvenilmez |
| Audit WAL unwritable | Protected execution durmalı | Evidence olmadan kritik write yapılmamalı |
| Approval store corrupt | Resolve/retry durmalı | Onay durumu güvenilmez |
| Parser failure | BLOCK | Güvenli sınıflandırılamayan SQL çalışmamalı |
| Multi-statement | BLOCK | Gizli ikinci operasyon riski |
| DB timeout | Execution state unknown raporlanmalı | Protected write otomatik retry edilmemeli |
| Result evidence commit failure | Kritik integrity failure | Execution olmuş ama evidence eksik |

## Dürüst raporlama

Fail-safe davranışın önemli parçası, belirsizliği saklamamaktır. AXIS "başarılı" diyemiyorsa bunu açıkça raporlamalıdır:

- `execution_state: unknown`
- `result_evidence_commit_failed`
- `audit_storage_unavailable`
- `approval_requires_manual_review`

Bu alanlar operatöre incident veya manuel inceleme gerekip gerekmediğini gösterir.

# Policy Yaşam Döngüsü ve Bütünlük

Policy dosyası production write-path davranışını belirler. Bu yüzden policy değişikliği normal config değişikliği gibi değil, güvenlik açısından kritik bir operasyon gibi ele alınmalıdır.

## Manifest-authoritative startup

Mevcut v0.9 modelinde `policies/policy_manifest.json` başlangıç için authoritative kaynaktır. Manifest şu bilgileri taşır:

- `manifest_version`
- `active_policy`
- `policy_id`
- `policy_version`
- `sha256`
- opsiyonel environment ve rollback metadata

AXIS, active policy dosyasını policy dizinine göre çözer ve path traversal gibi durumları reddeder.

## SHA-256 doğrulaması

Manifest içindeki `sha256`, active policy dosyasının raw byte içeriği üzerinden hesaplanır. AXIS startup sırasında dosyayı okur, SHA-256 hesaplar ve manifest ile karşılaştırır.

Bu kontrol şunu sağlar:

- yanlış dosya yüklenmesi yakalanır,
- local policy dosyasında beklenmeyen değişiklik yakalanır,
- policy_version mismatch yakalanır.

Bu kontrol şunu sağlamaz:

- remote attestation,
- asymmetric signature,
- distributed consensus,
- HSM/KMS backed trust.

## Startup policy validation

Başlangıçta:

1. manifest okunur,
2. path doğrulanır,
3. SHA-256 kontrol edilir,
4. JSON schema ve semantic validation yapılır,
5. policy version manifest ile karşılaştırılır,
6. activation dry-run corpus çalışır,
7. başarılıysa policy metadata runtime kararlarına taşınır.

```mermaid
flowchart TD
  A[Policy manifest] --> B[Path doğrulama]
  B --> C[SHA-256 hesapla]
  C --> D{Manifest ile eşleşiyor mu?}
  D -->|Hayır| X[Startup fail-fast]
  D -->|Evet| E[Schema ve semantic validation]
  E --> F{Dry-run geçti mi?}
  F -->|Hayır| X
  F -->|Evet| G[Policy aktif]
  G --> H[Kararlara policy metadata ekle]
```

## Dry-run

Dry-run, policy'nin temsilci SQL corpus üzerinde ne karar vereceğini gösterir. Önemli nokta: dry-run execution yapmaz, audit event yazmaz ve approval oluşturmaz.

Başlangıç dry-run corpus içinde safe read, scoped update, unsafe delete, bulk update/delete, DDL, unsupported SQL ve multi-statement gibi örnekler bulunur.

## Activation

Policy lifecycle endpoint'leri candidate policy oluşturma, validation, diff, dry-run, activation ve rollback akışlarını destekler. Mutating lifecycle endpoint'leri operator token gerektirir.

Activation için beklenen güvenlik kontrolleri:

- candidate mevcut olmalı,
- expected hash eşleşmeli,
- policy tekrar validate edilmeli,
- eski policy sessizce silinmemeli,
- audit evidence yazılmalı.

## Rollback

Rollback, önceki valid policy versiyonuna dönme mekanizmasıdır. Rollback de write-path davranışını değiştirdiği için security-sensitive kabul edilmelidir.

## Reload

Mevcut kaynakta manifest reload fonksiyonu vardır; ancak HTTP reload endpoint'i açılmamıştır ve `AXIS_ENABLE_POLICY_RELOAD=false` varsayılanı ile disabled durumdadır. Bu nedenle reload kavramı implementation içinde kontrollü gelecek yol olarak düşünülmelidir, genel unauthenticated runtime reload olarak sunulmamalıdır.

## Eski policy neden korunmalı?

Reviewer veya incident responder şu soruları sorabilir:

- Hangi policy ile karar verildi?
- O policy daha sonra değişti mi?
- Önceki policy'ye dönmek mümkün mü?
- Policy değişikliği write-path riskini artırdı mı?

Eski policy kayıtları bu yüzden sadece temizlik yükü değildir; audit ve rollback için gereklidir.

## Policy metadata propagation

Karar response'larında ve audit evidence içinde şu metadata bulunmalıdır:

- `policy_id`
- `policy_version`
- `policy_sha256`

Bu alanlar decision, approval ve evidence zincirinin aynı policy state'e bağlanmasını sağlar.

# Güven Sınırları

AXIS'in güvenlik modeli, hangi bileşenin trusted, hangi girdinin untrusted olduğunu açıkça ayırmayı gerektirir. Bu ayrım yapılmazsa doğru çalışan AXIS bile yanlış deployment nedeniyle zayıflar.

## Genel trust boundary

```mermaid
flowchart LR
  U[Untrusted caller input] --> A[AXIS trusted runtime]
  A --> P[Policy dosyaları sensitive config]
  A --> W[Audit WAL evidence source]
  A --> D[(Production PostgreSQL protected target)]
  B[Browser] --> CP[Control Plane server proxy]
  CP --> A
  CP -. server-side token .-> A
  B -. operator token görmemeli .-> CP
```

## Trusted kabul edilenler

Deployment varsayımı altında trusted kabul edilenler:

- AXIS process host
- AXIS runtime binary ve config
- active policy manifest ve policy dosyaları
- audit WAL dosyasına append yapan AXIS process
- approval SQLite store
- server-side Control Plane proxy
- PostgreSQL credentials'in AXIS tarafında güvenli tutulması

Bu trust, sınırsız değildir. Host compromise olursa local WAL ve policy dosyaları da risk altındadır. AXIS local manifest SHA-256 ile remote attestation iddiası yapmaz.

## Untrusted kabul edilenler

Untrusted veya doğrulanması gereken girdiler:

- request body içindeki actor/app/tenant/role/env alanları
- raw SQL
- SQL parameters
- browser input
- client tarafında gelen backend URL
- browser tarafında taşınan operator token
- AXIS'i bypass eden doğrudan DB path

JWT trusted context etkinse AXIS identity alanlarını token'dan türetebilir; aksi halde body alanları deployment tarafından doğrulanmadıkça spoofable kabul edilmelidir.

## Browser operator token görmemeli

Control Plane browser kodu `AXIS_OPERATOR_TOKEN` veya `AXIS_BACKEND_URL` almamalıdır. Mevcut Control Plane, `/api/axis/...` server-side proxy kullanır. Proxy backend URL'yi server environment'tan okur ve operator token'ı gerekli endpoint'lere server-side ekler.

Bu sınır kritik önemdedir:

- Browser token görürse token sızıntı yüzeyi büyür.
- Browser backend URL kontrol ederse SSRF veya yanlış backend routing riski oluşur.
- Client sadece local proxy path çağırmalıdır.

## Control Plane proxy sınırı

Control Plane proxy:

- route allowlist uygular,
- path segment validation yapar,
- JSON content-type bekler,
- backend URL'yi server-side config'ten alır,
- token'ı server-side ekler,
- hassas detayları sanitize eder.

Bu proxy full IAM değildir. Operator identity, RBAC, SSO ve mTLS gibi kontroller production deployment'ta ayrıca tasarlanmalıdır.

## Database protected target

Production PostgreSQL protected target'tır. AXIS'ten geçmeyen direct write trafiği AXIS tarafından kontrol edilemez.

Bu yüzden production ortamda:

- uygulamalar doğrudan write-capable DB credential almamalı,
- network policy AXIS dışı write path'i kapatmalı,
- DB role separation yapılmalı,
- bypass monitoring olmalı,
- backup ve restore ayrı güvenlik kontrolü olarak korunmalı.

## Audit WAL'ın rolü

Audit WAL evidence source'tur. Runtime logs veya JSONL projection bu rolü devralmaz.

WAL için beklenen sorumluluklar:

- append path korunmalı,
- disk ve volume güvenilirliği sağlanmalı,
- backup/retention planı olmalı,
- sadece trusted process yazabilmeli,
- verification sonuçları düzenli izlenmeli.

## Policy dosyaları sensitive security configuration

Policy dosyası production write-path davranışını değiştirir. Bu nedenle:

- review edilmeden değiştirilmemeli,
- manifest SHA-256 güncellemesi bilinçli yapılmalı,
- rollback yolu korunmalı,
- candidate diff ve dry-run kullanılmalı,
- policy change audit edilmeli.

# Runtime Akışları

Bu dosya, AXIS'in temel runtime akışlarını ayrı ayrı gösterir. Diyagramlar basit tutulmuştur; amaç reviewer'ın execution öncesi karar ve evidence bağlantısını hızlıca görmesidir.

## 1. Safe READ allowed

Safe read, policy default veya explicit allow ile çalışabilir. Yine de request ve decision evidence üretilebilir.

```mermaid
flowchart TD
  A[Safe SELECT isteği] --> B[Parse ve classify READ]
  B --> C[Policy evaluate]
  C -->|ALLOW| D[Decision evidence]
  D --> E[PostgreSQL execution]
  E --> F[Result evidence]
  F --> G[Response 200]
```

## 2. Safe WRITE allowed

Scoped write, policy'de explicit allow varsa çalışabilir. Protected write için decision evidence execution öncesinde commit edilmelidir.

```mermaid
flowchart TD
  A[Scoped WRITE isteği] --> B[Scope ve target çıkar]
  B --> C[Policy explicit allow]
  C --> D[Audit decision commit]
  D --> E[Database adapter]
  E --> F[PostgreSQL]
  F --> G[Execution result evidence]
```

## 3. Risky WRITE requiring approval

Approval gereken write ilk istekte çalışmaz.

```mermaid
flowchart TD
  A[Riskli WRITE] --> B[Classifier risk sinyalleri]
  B --> C[Policy evaluate]
  C -->|APPROVAL_REQUIRED| D[Approval created evidence]
  D --> E[PENDING approval]
  E --> F[Response 202 approval_id]
  F --> G[Execution yok]
```

## 4. BLOCK flow

Block kararı PostgreSQL'e gitmez.

```mermaid
flowchart TD
  A[Tehlikeli veya geçersiz SQL] --> B[Parser veya policy]
  B --> C[BLOCK]
  C --> D[Block evidence]
  D --> E[Execution yok]
  E --> F[Response 4xx]
```

## 5. Approval resolve flow

Approve/reject final state üretir. Approve execution değildir; caller aynı request ile retry yapar.

```mermaid
flowchart TD
  A[PENDING approval] --> B[Operator resolve]
  B --> C{Karar}
  C -->|Approve| D[APPROVED state]
  C -->|Reject| E[REJECTED state]
  D --> F[Resolved evidence]
  E --> F
  F --> G[Final state immutable]
```

## 6. Audit verification flow

Verification WAL'ı okur, event hash ve previous hash zincirini kontrol eder.

```mermaid
flowchart TD
  A[Audit WAL] --> B[Event satırlarını oku]
  B --> C[event_hash recompute]
  C --> D[previous_hash kontrol]
  D --> E{Tutarlı mı?}
  E -->|Evet| F[verified]
  E -->|Hayır| G[tampered veya unverifiable]
```

## 7. Startup policy integrity flow

Startup policy bütünlüğü doğrulanmadan trafik kabul edilmemelidir.

```mermaid
flowchart TD
  A[Başlangıç] --> B[Audit WAL continuity]
  B --> C[Policy manifest oku]
  C --> D[Policy SHA-256 doğrula]
  D --> E[Schema validation]
  E --> F[Activation dry-run]
  F --> G{Geçti mi?}
  G -->|Evet| H[Listener trafik kabul eder]
  G -->|Hayır| I[Fail-fast]
```

## 8. Corrupt audit fail-fast flow

Audit WAL bozuksa sistem güvenli başlamamalıdır.

```mermaid
flowchart TD
  A[AXIS başlıyor] --> B[Audit WAL oku]
  B --> C{Hash zinciri geçerli mi?}
  C -->|Evet| D[Son hash ile devam et]
  C -->|Hayır| E[Startup fail-fast]
  E --> F[Protected traffic yok]
```

## 9. Evidence export flow

Evidence export, seçilmiş event'leri ve verification metadata'yı taşınabilir hale getirir.

```mermaid
flowchart TD
  A[Audit WAL] --> B[Filtrele ve event seç]
  B --> C[Verification metadata ekle]
  C --> D[Policy metadata ekle]
  D --> E[Bundle üret]
  E --> F[Offline verification için paylaş]
```

# Hata Modu Matrisi

Bu matris, reviewer'ın AXIS davranışını hata senaryoları altında değerlendirmesi için hazırlanmıştır. Amaç sadece "hata oldu" demek değil; hatanın write-path güvenliği, evidence ve operatör aksiyonu açısından ne anlama geldiğini göstermektir.

| Hata senaryosu | Risk | AXIS'in beklenen davranışı | Üretilen evidence | Neden önemli? |
|---|---|---|---|---|
| Postgres down | Allowed query çalışmayabilir; write sonucu belirsiz olabilir | Startup connection check başarısızsa başlama hatası; runtime'da structured DB error | Execution failed veya DB error evidence, mümkünse response evidence | DB hatası policy allow gibi gösterilmemeli |
| Audit unwritable | Kritik karar kanıtlanamaz | Protected write execution öncesi fail-closed | `audit_storage_unavailable` reason; WAL yazılamıyorsa event olmayabilir, runtime safe summary olabilir | Evidence olmadan production write güvenilir değildir |
| Invalid policy | Yanlış enforcement | Startup fail-fast veya candidate rejection | Policy activation failed event yazılabiliyorsa yazılır | Permissive fallback tehlikelidir |
| Policy checksum mismatch | Policy dosyası manifest ile uyumsuz | Startup fail-fast | Activation failure evidence mümkünse | Local policy bütünlüğü bozulmuş olabilir |
| Corrupt audit WAL/log | Evidence zinciri güvenilmez | Startup fail-fast; verification tampered/unverifiable raporlar | Verification error veya startup hata | Bozuk evidence üzerine güvenli runtime inşa edilmemeli |
| Corrupt approval store | Yanlış approve/reject veya replay riski | Store açılışı/resolve fail-safe hata üretmeli | Approval resolve error; varsa runtime safe summary | Approval state güvenilir değilse execution yapılmamalı |
| Malformed request | Parser veya validator bypass riski | Request rejection, structured error | Request rejected / decision made / blocked evidence mümkünse | Kötü input generic panic'e dönüşmemeli |
| Huge payload | Memory/latency baskısı | Body veya SQL size limit ile reddet | `request_body_too_large` veya `sql_too_large`; oversized SQL raw yazılmaz | Resource abuse ve secret leak riski azalır |
| Parser failure | Tehlikeli SQL güvenli sanılabilir | `BLOCK` / fail-closed | `parser_error`, `unsupported_sql_shape` veya ilgili reason | Unsupported şekil allow olmamalı |
| Concurrent approval race | Aynı approval birden fazla execution için kullanılabilir | Tek reservation başarılı olmalı; diğerleri block | Approval retry blocked veya execution state evidence | Single-use approval garantisi kritik |
| Restart during traffic | Prepared state veya audit continuation kaybolabilir | Audit chain son hash'ten devam etmeli; in-memory prepared state kaybolunca unresolved execute block olmalı | Restart sonrası previous_hash continuity; unresolved prepared evidence | Restart güvenlik modelini zayıflatmamalı |
| DB pool pressure | Request'ler bekler veya panic riski | Bounded acquire timeout, structured `db_pool_exhausted` | Execution failed / response evidence mümkünse | Saturation güvenli hata olmalı |
| Evidence commit failure | Execution olmuş ama sonuç evidence yazılamamış olabilir | Execution öncesiyse block; execution sonrasıysa critical integrity failure raporla | `result_evidence_commit_failed`, integrity state | Sessiz başarı yanlış güven yaratır |
| DB timeout | PostgreSQL query sonucu unknown olabilir | `execution_state: unknown`; protected write otomatik retry edilmemeli | `execution_unknown` evidence mümkünse | Timeout sonrası yeniden deneme veri tutarsızlığı yaratabilir |
| Policy reload failure | Yanlış veya yarım policy state | Eski active policy korunmalı | Reload failure evidence mümkünse | Policy değişimi atomik ve güvenli olmalı |
| Direct DB bypass | AXIS hiç karar veremez | AXIS bunu runtime içinde engelleyemez; deployment kapatmalı | AXIS evidence üretmez çünkü trafik AXIS'e gelmez | En önemli deployment boundary budur |

## Reviewer notu

Bir hata senaryosu güvenli sayılabilmek için şu üç soruya cevap vermelidir:

- Production write PostgreSQL'e ulaştı mı?
- Ulaştıysa sonucu kesin mi, belirsiz mi?
- Bu durum audit evidence ile dürüstçe raporlandı mı?

# Reviewer Rehberi

Bu rehber, dış teknik reviewer'ın AXIS'i iddia, kaynak ve evidence ayrımıyla incelemesi için hazırlanmıştır.

## İlk neye bakmalı?

1. Protected write path gerçekten AXIS'ten geçiyor mu?
2. `/query` akışı execution öncesi parser/classifier ve policy evaluate yapıyor mu?
3. `ALLOW`, `BLOCK`, `REQUIRE_APPROVAL` davranışları response ve audit event'lerde tutarlı mı?
4. Approval gereken request ilk istekte PostgreSQL'e gitmiyor mu?
5. Audit WAL hash-chain verification çalışıyor mu?
6. Policy manifest SHA-256 mismatch startup'ı durduruyor mu?
7. Control Plane browser'a backend URL veya operator token sızdırıyor mu?
8. Limitler açıkça yazılmış mı?

## Hangi claim nasıl doğrulanır?

| Claim | Doğrulama yolu |
|---|---|
| SQL execution öncesi policy kararı var | `src/gate/listener.rs`, `src/gate/enforcer.rs`, `/query` testleri |
| Dangerous delete block olur | Policy fixture ve regression testleri, `/query` örnekleri |
| Approval ilk request'te execution yapmaz | Approval response `NOT_EXECUTED`, audit event, DB state kontrolü |
| Approved retry exact context ister | `ApprovalRetryProof`, SQL fingerprint ve policy metadata eşleşme kontrolleri |
| WAL hash chain doğrulanır | `/evidence/verify`, `/audit/verify`, `src/audit/verifier.rs` |
| Policy integrity startup'ta kontrol edilir | `src/policy/manifest.rs`, checksum mismatch testi |
| Browser token görmez | `control-plane/src/app/api/axis/[...path]/route.ts`, client proxy çağrıları |

## AXIS nasıl zorlanır?

Reviewer şu inputları denemelidir:

- `DELETE FROM users`
- `DELETE FROM orders WHERE status='old'`
- `UPDATE orders SET status='x'`
- `UPDATE orders SET status='x' WHERE id=1 AND tenant_id='acme'`
- `DROP TABLE users`
- `SELECT 1; DELETE FROM users`
- `VACUUM`
- malformed SQL
- oversized SQL
- unknown prepared `EXECUTE`
- missing `session_id` ile `PREPARE` veya `EXECUTE`
- aynı approval id ile paralel retry
- policy checksum mismatch
- audit WAL tamper

## Kritik davranışlar

- Block edilen SQL PostgreSQL'e ulaşmamalı.
- Approval-required SQL ilk request'te çalışmamalı.
- Reject edilen approval execution'a dönüşmemeli.
- Approved retry sadece aynı bağlamla çalışmalı.
- Audit write failure protected write'ı durdurmalı.
- Result evidence commit failure sessiz başarı olmamalı.
- Unsupported SQL güvenli tarafta kalmalı.

## Anlamlı testler

- `cargo test`
- parser bypass case corpus
- policy case corpus
- approval race testleri
- audit restart continuity testleri
- chaos/failure-mode scriptleri
- `/evidence/verify` ve offline evidence verification
- Control Plane build/typecheck/e2e real mode testleri

## Sorulması gereken sorular

- Production deployment'ta direct DB bypass nasıl engellenecek?
- Request identity gerçekten nereden geliyor?
- Operator token nasıl saklanıyor ve rotate ediliyor?
- Policy değişikliklerini kim review ediyor?
- Audit WAL nerede tutuluyor, nasıl yedekleniyor?
- Evidence export imzalı mı, imzasızsa bu açıkça raporlanıyor mu?
- Native wire protocol gerekli mi, yoksa HTTP adapter yeterli mi?
- Failure durumunda operatör runbook'u ne diyor?

## Red flag'ler

- "AXIS her şeyi çözer" gibi genel iddia.
- Direct DB bypass limitinin saklanması.
- Runtime logların audit evidence gibi sunulması.
- Approval reject için evidence olmaması.
- Policy checksum mismatch'e rağmen servis başlaması.
- Browser'ın `AXIS_OPERATOR_TOKEN` veya backend URL görmesi.
- Unsupported SQL'in allow edilmesi.
- Evidence chain bozulduğunda sessiz repair yapılması.

## İyi işaretler

- Claim'lerin kaynak dosya ve testlerle bağlanması.
- `policy_id`, `policy_version`, `policy_sha256` alanlarının karar ve evidence içinde bulunması.
- Approval resolve sonrası immutable final state.
- Audit WAL'ın source of truth olarak ele alınması.
- JSONL projection ve runtime logs için sınırlı rol tanımı.
- Limitlerin açık ve görünür yazılması.
- Fail-closed davranışın testlerle desteklenmesi.

## Limitler nerede?

Limitlerin merkezi özeti [14-HEDEF-OLMAYANLAR-VE-LIMITLER.md](14-HEDEF-OLMAYANLAR-VE-LIMITLER.md) dosyasındadır. Reviewer, herhangi bir güçlü claim gördüğünde bu limitlerle karşılaştırmalıdır.

# Müşteri İçin Teknik Anlatım

AXIS, production veritabanı operasyonlarını çalışmadan önce policy ile değerlendiren bir kontrol katmanıdır. Amaç, yetkili sistemlerden gelen riskli write işlemlerini execution öncesinde durdurmak, onaya bağlamak veya kontrollü şekilde geçirmek ve bu kararları audit evidence ile kanıtlanabilir hale getirmektir.

## Neden önemlidir?

Production veritabanı riskleri sadece dış saldırıdan gelmez. Yanlış migration, hatalı admin script'i, geniş kapsamlı ORM query'si, internal tool bug'ı veya AI agent tarafından üretilen SQL de veri kaybına veya yanlış değişikliğe neden olabilir.

AXIS bu operasyonel riski azaltır:

- SQL'i çalışmadan önce inceler,
- write/delete/DDL risklerini sınıflandırır,
- policy kararı üretir,
- approval gereken işleri beklemeye alır,
- karar ve sonucu audit evidence olarak kaydeder.

## Altyapıda nereye oturur?

```mermaid
flowchart LR
  App[Uygulama ve servisler] --> Axis[AXIS control layer]
  Tool[Internal tool ve script] --> Axis
  Agent[AI agent] --> Axis
  Axis -->|ALLOW| Db[(Production PostgreSQL)]
  Axis -->|BLOCK| Stop[Execution yok]
  Axis -->|APPROVAL_REQUIRED| Approval[Onay süreci]
  Axis --> Audit[(Audit evidence)]
  Ops[Control Plane] --> Axis
```

Mevcut repo, HTTP `/query` adapter modelini gösterir. Native PostgreSQL wire protocol desteği ayrı değerlendirilmesi gereken bir konudur.

## Execution öncesinde ne yapar?

AXIS şu kontrolleri yapar:

- request validation,
- SQL parse ve classification,
- operation, target, scope ve risk signal çıkarımı,
- active policy evaluation,
- `ALLOW`, `BLOCK` veya `REQUIRE_APPROVAL` kararı,
- protected write için audit decision evidence commit.

`BLOCK` veya `REQUIRE_APPROVAL` durumunda SQL ilk request'te PostgreSQL'e ulaşmaz.

## Execution sonrasında neyi kanıtlar?

AXIS audit evidence ile şunları göstermeyi hedefler:

- hangi request geldi,
- SQL fingerprint neydi,
- hangi policy versiyonu kullanıldı,
- hangi rule match etti,
- karar neydi,
- approval varsa hangi approval id ile bağlıydı,
- execution sonucu veya belirsizlik neydi,
- event hash zinciri tutarlı mı.

Bu kanıt modeli normal uygulama loglarından farklıdır; WAL ve hash-chain doğrulamasına dayanır.

## Operasyonel değer

AXIS şu alanlarda değer sağlar:

- destructive SQL riskinin azaltılması,
- risky write için explicit approval,
- policy değişikliklerinin daha kontrollü yürütülmesi,
- incident review için karar kanıtı,
- uygulama ve operatör hatalarının execution öncesinde yakalanması,
- customer pilotlarında protected write path'in gözlemlenmesi.

## Ne vaat etmez?

AXIS şu iddiaları yapmamalıdır:

- bütün database security kontrollerinin yerini alır,
- IAM/RBAC gereksizdir,
- backup ve monitoring gereksizdir,
- direct DB bypass mümkün değildir,
- compliance certification sağlar,
- local SHA-256 remote attestation demektir,
- her ORM veya her SQL shape otomatik desteklenir,
- native PostgreSQL wire protocol coverage mevcut HTTP adapter ile kanıtlanmıştır.

AXIS doğru deployment ve operator discipline ile anlam kazanır. Özellikle production'da database credentials, network path ve role separation AXIS güvenlik modelinin dış ama zorunlu parçalarıdır.

# Hedef Olmayanlar ve Limitler

Bu dosya AXIS'in ne iddia etmediğini açıkça yazar. Bu sınırlar zayıflık saklamak için değil, doğru teknik değerlendirme için gereklidir.

## AXIS'in iddia etmediği şeyler

AXIS:

- bütün database security kontrollerinin yerine geçmez,
- IAM/RBAC yerine tek başına geçmez,
- backup, monitoring, network security veya least privilege yerine geçmez,
- WAF değildir,
- AI agent değildir,
- compliance certification sağlamaz,
- her SQL dialect için tam coverage iddia etmez,
- native PostgreSQL wire protocol desteğini mevcut HTTP adapter ile kanıtlamaz,
- direct DB bypass trafiğini kendi içinde engelleyemez,
- remote attestation veya hardware-backed trust sağlamaz,
- production-ready HA mimarisi iddia etmez.

## Local manifest SHA-256 limiti

Policy manifest içindeki SHA-256, local policy dosyasının manifest ile uyumlu olduğunu gösterir. Bu önemlidir ama sınırı açıktır:

- asymmetric signing değildir,
- remote attestation değildir,
- KMS/HSM backed key trust değildir,
- multi-instance consensus değildir,
- host compromise riskini ortadan kaldırmaz.

## Native database wire protocol limiti

Mevcut repo, HTTP `/query` adapter ve uygulama entegrasyonu modelini gösterir. Native PostgreSQL wire protocol veya transparent drop-in proxy desteği ayrı implementation ve ayrı review gerektirir.

Bu özellikle müşteri entegrasyonlarında net söylenmelidir.

## Direct DB bypass limiti

AXIS sadece kendisinden geçen operasyonları kontrol eder. Eğer bir servis doğrudan PostgreSQL'e write-capable credential ile bağlanabiliyorsa AXIS bu write'ı göremez.

Production deployment'ta şunlar gerekir:

- private DB network,
- firewall/security group,
- DB role separation,
- credential discipline,
- uygulamaların write-capable direct credential almaması,
- bypass monitoring.

## Policy kalitesi limiti

AXIS policy kadar iyi karar verir. Kötü yazılmış veya fazla permissive policy, AXIS'in güvenlik değerini azaltır.

Bu yüzden:

- policy review gerekir,
- candidate diff gerekir,
- dry-run gerekir,
- unsafe default allow risklidir,
- policy değişikliği audit edilmelidir.

## Deployment trust limiti

AXIS runtime host, audit storage ve policy dosyaları güven sınırı içindedir. Bu host veya storage kontrol dışına çıkarsa local evidence güveni zayıflar.

Bu nedenle production planı:

- hardened host,
- restricted filesystem permissions,
- backup ve retention,
- secret management,
- monitored audit path,
- incident runbook içermelidir.

## Approval limiti

Approval, ilk request'i bekletir ve execution yapmaz. Approved durumunda execution için aynı request context ve `approval_id` ile retry gerekir. Bu model uzun süre açık database transaction tutmaz.

Bu iyi bir güvenlik/operasyon tradeoff'u olabilir, fakat uygulama entegrasyonunun retry semantics'i doğru uygulaması gerekir.

## Prepared statement limiti

Mevcut model AXIS-side prepared intent tracking yapar. Database-side PostgreSQL prepared statement connection affinity iddiası yapmaz. In-memory session state restart sonrası kaybolur; eski `EXECUTE` fail-safe block olur.

## Evidence signing limiti

Repo evidence bundle signing için Ed25519 desteği içerir. Ancak signing etkin değilse evidence export bunu imzasız olarak raporlamalıdır. İmzalı bundle bile host ve key management güvenlik varsayımlarını ortadan kaldırmaz.

## Performans limiti

AXIS execution path'e ek kontrol ve audit maliyeti getirir. Benchmark sonuçları müşteri workload'u için otomatik garanti değildir. Pilotlar gerçek workload, latency toleransı ve failure mode gereksinimleriyle test edilmelidir.

# Sözlük

| Terim | Türkçe açıklama |
|---|---|
| deterministic | Deterministik. Aynı input ve aynı policy ile aynı kararın üretilmesi. AI tahmini veya rastlantısal yorum değildir. |
| policy | Politika. Hangi SQL operasyonunun allow, block veya approval gerektirdiğini belirleyen güvenlik kuralları. |
| execution | Çalıştırma. SQL'in PostgreSQL'e gönderilip veritabanında uygulanması veya sonuç üretmesi. |
| audit evidence | Denetlenebilir kanıt. Karar, context, policy metadata ve hash-chain bilgisi taşıyan güvenlik kaydı. |
| WAL | Write-ahead log. AXIS'te audit evidence için source of truth olarak kullanılan append-only kayıt dosyası. |
| JSONL projection | JSON Lines projection. WAL commit sonrası operator kolaylığı için yazılan türetilmiş JSON satırları. WAL'ın yerine geçmez. |
| hash chain | Hash zinciri. Her event'in bir önceki event hash'ine bağlanması. |
| event_hash | Event içeriğinden hesaplanan SHA-256 hash. Event değişirse hash doğrulaması bozulur. |
| previous_hash | Önceki event'in `event_hash` değeri. Zincir sürekliliğini sağlar. |
| approval | Onay. Riskli işlemin ilk istekte çalışmaması ve operatör kararı gerektirmesi. |
| fail-safe | Hata durumunda güvenli tarafa geçme prensibi. |
| fail-closed | Belirsizlik veya kritik hata durumunda execution yolunun kapalı kalması. |
| manifest | Active policy dosyasını, policy kimliğini, versiyonunu ve SHA-256 değerini tanımlayan dosya. |
| SHA-256 | Kriptografik hash algoritması. AXIS policy integrity ve event hash hesaplarında kullanılır. |
| parser/classifier | SQL'i parse edip read/write/DDL, target, scope ve risk sinyallerini çıkaran katman. |
| control plane | Operatör arayüzü. AXIS backend'e server-side proxy üzerinden bağlanan Next.js UI. |
| operator token | Operatör yetkilendirme token'ı. Approval resolve ve policy mutation gibi endpoint'lerde kullanılır; browser'a verilmemelidir. |
| trust boundary | Güven sınırı. Hangi bileşenin trusted, hangi girdinin untrusted olduğunu ayıran mimari çizgi. |
| production write path | Production verisini değiştiren write/delete/DDL operasyonlarının geçtiği kritik execution yolu. |
| `ALLOW` | Policy'nin execution'a izin verdiği karar. |
| `BLOCK` | Policy, parser veya fail-safe mekanizmanın execution'ı durdurduğu karar. |
| `APPROVAL_REQUIRED` | Kavramsal approval kararı. Mevcut API'de çoğunlukla `REQUIRE_APPROVAL` adıyla görünür. |
| `SUSPEND` | Kavramsal askıya alma durumu. Mevcut policy enum'u değildir; approval bekleme veya manual review gibi durumları anlatır. |
| `REJECT` | Approval resolve kararında reddetme. Store durumunda `REJECTED`, final execution kararında block olarak yansır. |

# Kontrol Listesi

Bu checklist, açıklama paketinin amacına ulaşıp ulaşmadığını kontrol etmek için kullanılabilir.

## Anlama kontrolü

- [ ] Okuyucu AXIS'i tek cümlede anlatabiliyor mu?
- [ ] Okuyucu AXIS'in ne olmadığını açıkça görebiliyor mu?
- [ ] Okuyucu mimariyi kabaca çizebiliyor mu?
- [ ] Okuyucu policy-before-execution modelini anlayabiliyor mu?
- [ ] Okuyucu traditional execution sonrası log modeli ile AXIS modelini ayırabiliyor mu?
- [ ] Okuyucu `ALLOW`, `BLOCK`, `REQUIRE_APPROVAL` davranışlarını ayırabiliyor mu?
- [ ] Okuyucu `APPROVAL_REQUIRED` kavramsal adı ile API'deki `REQUIRE_APPROVAL` adını karıştırmadan anlayabiliyor mu?

## Evidence kontrolü

- [ ] Okuyucu audit evidence modelini anlayabiliyor mu?
- [ ] Okuyucu WAL'ın source of truth olduğunu görebiliyor mu?
- [ ] Okuyucu JSONL projection ve runtime logs'un sınırlı rolünü anlayabiliyor mu?
- [ ] Okuyucu `event_hash` ve `previous_hash` ilişkisini açıklayabiliyor mu?
- [ ] Okuyucu restart continuity'nin neden önemli olduğunu anlayabiliyor mu?
- [ ] Okuyucu evidence export'un ne işe yaradığını ve neyi kanıtlamadığını görebiliyor mu?

## Approval kontrolü

- [ ] Okuyucu approval flow'u anlayabiliyor mu?
- [ ] Pending, approved, rejected ve retry ayrımı net mi?
- [ ] Reject durumunda bile evidence gerekliliği açıklanmış mı?
- [ ] Final approval state'in immutable olması anlaşılır mı?

## Fail-safe kontrolü

- [ ] Okuyucu fail-safe davranışı anlayabiliyor mu?
- [ ] Policy invalid olduğunda startup fail-fast gerektiği açık mı?
- [ ] Audit write failure durumunda protected execution'ın durması gerektiği açık mı?
- [ ] Result evidence commit failure'ın dürüstçe raporlanması gerektiği açık mı?

## Reviewer kontrolü

- [ ] Reviewer claim/evidence ayrımını görebiliyor mu?
- [ ] Hangi claim'in hangi kaynak veya endpoint ile doğrulanacağı yazılmış mı?
- [ ] Red flag ve good sign listeleri yeterince somut mu?
- [ ] Failure-mode matrix ciddi ve reviewer-friendly mi?

## Limit kontrolü

- [ ] Limitler açıkça yazılmış mı?
- [ ] Direct DB bypass sınırı saklanmamış mı?
- [ ] Native wire protocol limitleri açık mı?
- [ ] Local manifest SHA-256'ın remote attestation olmadığı yazılmış mı?
- [ ] IAM, backup, monitoring, network security ve least privilege gerekliliği korunmuş mu?
- [ ] Marketing veya abartılı iddia var mı?

# AXIS Açıklama Paketi v1

Bu klasör, VARUX AXIS'in mevcut repo davranışına dayalı sade ve teknik Türkçe açıklama paketidir. Amaç; bir teknik reviewer'ın, mühendis adayının, müşteri adayının veya kurucunun AXIS'in ne yaptığını, ne yapmadığını, hangi güvenlik modeline dayandığını ve hangi sınırlara sahip olduğunu hızlıca anlayabilmesidir.

Bu paket reklam metni değildir. Mevcut runtime/source dosyalarına dokunmadan, ayrı bir dokümantasyon katmanı olarak hazırlanmıştır.

## AXIS tek cümlede

AXIS, production veritabanı operasyonlarını çalışmadan önce politika ile değerlendiren ve her kritik karar için doğrulanabilir audit evidence üreten deterministik bir kontrol katmanıdır.

## 5 dakikada AXIS

AXIS'i production PostgreSQL önünde çalışan kontrollü bir karar katmanı gibi düşünün:

1. Uygulama, internal tool, script, AI agent veya servis AXIS'e SQL isteği gönderir.
2. AXIS isteği doğrular, SQL'i parse eder ve sınıflandırır.
3. Policy engine, aktif policy'ye göre `ALLOW`, `BLOCK` veya `REQUIRE_APPROVAL` kararı üretir.
4. `ALLOW` ise işlem PostgreSQL'e gönderilir.
5. `BLOCK` ise işlem PostgreSQL'e ulaşmaz.
6. `REQUIRE_APPROVAL` ise approval kaydı oluşur; işlem onay ve aynı bağlamla tekrar gönderim olmadan çalışmaz.
7. Kritik kararlar WAL tabanlı audit evidence olarak yazılır.
8. Evidence, `event_hash` ve `previous_hash` zinciri ile doğrulanabilir.

Not: Bu dokümanda `APPROVAL_REQUIRED` kavramsal ad olarak kullanılır. Mevcut Rust API'sinde aynı karar çoğunlukla `REQUIRE_APPROVAL` olarak serileşir; policy deserialization tarafında `APPROVAL_REQUIRED` alias'ı da desteklenir.

## Okuma sırası

| Sıra | Dosya | Ne anlatır? |
|---:|---|---|
| 1 | [00-AXIS-NEDIR.md](00-AXIS-NEDIR.md) | AXIS'in tanımı, çözdüğü problem ve ne olmadığı |
| 2 | [01-ZIHINSEL-MODEL.md](01-ZIHINSEL-MODEL.md) | Yeni başlayan için profesyonel zihinsel model |
| 3 | [02-POLICY-BEFORE-EXECUTION.md](02-POLICY-BEFORE-EXECUTION.md) | Execution öncesi policy kararının neden temel prensip olduğu |
| 4 | [03-MIMARI-HARITA.md](03-MIMARI-HARITA.md) | Ana mimari katmanlar ve sorumluluklar |
| 5 | [04-KARAR-AKISI.md](04-KARAR-AKISI.md) | SQL isteğinin karara dönüşmesi |
| 6 | [05-ONAY-AKISI.md](05-ONAY-AKISI.md) | Approval yaşam döngüsü |
| 7 | [06-AUDIT-EVIDENCE.md](06-AUDIT-EVIDENCE.md) | WAL, JSONL projection, hash chain ve export |
| 8 | [07-FAIL-SAFE-DAVRANIS.md](07-FAIL-SAFE-DAVRANIS.md) | Fail-safe ve fail-closed davranışlar |
| 9 | [08-POLICY-YASAM-DONGUSU-VE-BUTUNLUK.md](08-POLICY-YASAM-DONGUSU-VE-BUTUNLUK.md) | Manifest, SHA-256, dry-run, activation ve rollback |
| 10 | [09-GUVEN-SINIRLARI.md](09-GUVEN-SINIRLARI.md) | Trusted/untrusted sınırlar |
| 11 | [10-RUNTIME-AKISLARI.md](10-RUNTIME-AKISLARI.md) | Ayrı runtime akışları |
| 12 | [11-HATA-MODU-MATRISI.md](11-HATA-MODU-MATRISI.md) | Hata senaryoları ve beklenen davranışlar |
| 13 | [12-REVIEWER-REHBERI.md](12-REVIEWER-REHBERI.md) | Teknik reviewer için doğrulama rehberi |
| 14 | [13-MUSTERI-ICIN-TEKNIK-ANLATIM.md](13-MUSTERI-ICIN-TEKNIK-ANLATIM.md) | Müşteri adayına teknik anlatım |
| 15 | [14-HEDEF-OLMAYANLAR-VE-LIMITLER.md](14-HEDEF-OLMAYANLAR-VE-LIMITLER.md) | Açık non-goals ve limitler |
| 16 | [15-SOZLUK.md](15-SOZLUK.md) | Terimler sözlüğü |
| 17 | [CHECKLIST.md](CHECKLIST.md) | Paket kalite kontrol listesi |

## Diyagramlar

Mermaid kaynakları [diagrams/](diagrams/) altında ayrı dosya olarak tutulur. Tek başına açılabilir HTML görseli [visual/axis-map.html](visual/axis-map.html) içindedir ve harici dependency kullanmaz.

## Uygulandı, kavramsal, gelecek çalışma ayrımı

Bu paketteki anlatım mevcut repo içindeki Rust backend, policy lifecycle, approval store, audit WAL, evidence verification, Control Plane proxy ve pilot dokümanlarına göre yazılmıştır.

Kavramsal ifadeler özellikle işaretlenir. Örneğin `SUSPEND`, mevcut kaynakta ayrı bir policy enum değeri değildir; approval veya belirsiz execution durumlarında kullanılan askıya alma davranışını anlatan kavramsal bir terimdir. Native PostgreSQL wire protocol desteği de mevcut HTTP adapter modelinden ayrı değerlendirilmelidir.

# FIRST REVIEW - AXIS Ilk Mimari Paylasim Kaynagi

AXIS, uygulama ile PostgreSQL veritabani arasina girerek riskli veritabani islemlerini calismadan once kontrol eden ve izin verilmeyenleri durduran bir guvenlik katmanidir; uygulama normal veritabanina baglaniyormus gibi davranir, ancak korunan istekler once AXIS politikasindan gecer.

## Kanit Noktalari

- Real-driver test paketi 2026-07-05 tarihinde 37 test olarak gecmis gorunuyor.
- Dort ekosistem repo kanitlarinda yer aliyor: psycopg3, asyncpg, Prisma Client 6.19.3 ve PostgreSQL JDBC pgjdbc 42.7.7.
- Parametreli SELECT, INSERT, UPDATE ve policy tarafindan engellenen DELETE senaryolari bu dort ekosistemde test matrisi kapsaminda.
- Engellenen DELETE akislari SQLSTATE 42501 ile dogrulaniyor; baglanti sonraki islemler icin kullanilabilir kaliyor.
- Savepoint ile kismi kurtarma psycopg3, asyncpg, Prisma ve PostgreSQL JDBC icin test matrisinde kapsaniyor.

Bu materyal, AXIS'in mimarisini ve bugunku kanitli sinirlarini Gokay'a ilk kez tam mimari baglamiyla anlatmak icin hazirlanmis sunum kaynagidir.
# Problem Ve Neden Simdi

## Kurumsal Risk Sinifi

- Kritik veritabanlarinda en buyuk risklerden biri, yetkili kanaldan gelen ama is sonucu tehlikeli olan yazma islemleridir.
- Ornek risk siniflari:
  - Yanlis migration veya operasyon scripti.
  - Hatalı uygulama kodu veya ORM davranisi.
  - Ele gecirilmis uygulama kimlik bilgileri.
  - Iyi niyetli ama yanlis kapsamli toplu veri degisikligi.
- Bu islemler cogu zaman veritabanina gecerli kullanici, gecerli sifre ve gecerli ag yolu ile gelir.

## Mevcut Araclarin Goremeyebilecegi Katman

- Firewall, WAF ve klasik ag kontrolleri paketin nereye gittigini bilir; veritabani isteginin anlamini genellikle bilmez.
- Bir istegin "veri okuma" mi, "toplu veri silme" mi, "genis kapsamli degistirme" mi oldugunu anlamak SQL semantigi gerektirir.
- Uygulama loglari olaydan sonra faydalidir; ancak tehlikeli islemi calismadan once durdurmak icin yeterli olmayabilir.

## AXIS'in Hedefledigi Bosluk

- AXIS, veritabani isteginin calismadan hemen onceki noktasinda karar vermeyi hedefler.
- Amac, normal yetki zinciri icinden gelen ama riskli olan islemleri policy ile durdurmak veya ek onaya yonlendirmektir.
- Bu, mevcut IAM, veritabani yetkileri, ag segmentasyonu ve audit kontrollerinin yerine gecmez; onlarin onune ek bir son kontrol noktasi koyar.
# AXIS Nedir

AXIS, uygulama ile PostgreSQL arasina seffaf sekilde giren ve tehlikeli veritabani islemlerini calismadan once durduran bir politika katmanidir.

## Duz Anlatim

- Uygulama PostgreSQL'e baglaniyormus gibi davranir.
- AXIS, istegin ne yapmaya calistigini siniflandirir.
- Izinli istekler PostgreSQL'e ulasir.
- Izin verilmeyen istekler veritabanina gonderilmeden durdurulur.
- Uygulama, engellemeyi PostgreSQL uyumlu bir hata olarak gorur ve kendi hata veya geri alma mekanizmasini kullanabilir.

## Audit Ve Kanit Durumu

- Repo, yerel audit WAL kayitlari, hash zinciri alanlari ve dogrulama endpointleri icin testler iceriyor.
- Testlerde audit dogrulama, zincir bozulmasini "tampered" olarak raporlayabiliyor.
- Testlerde imzali export yapilandirildiginda, degistirilmis export bundle imza dogrulamasindan gecmiyor.
- Bu, yerel kanit butunlugu ve dogrulama davranisinin test edildigini gosterir.
- Bu repo, harici degistirilemez defter, KMS destekli imza dagitimi veya coklu instance audit konsensusu iddiasini kanitlanmis uretim ozelligi olarak sunmuyor.
# Mimari Ozet

## Akis

- Uygulama normal veritabani istemcisi gibi baglanir.
- AXIS, gelen istegi en erken guvenli noktada siniflandirir.
- Policy sonucu izinliyse istek PostgreSQL'e gider.
- Policy sonucu engelleme ise istek PostgreSQL'e hic ulasmaz.
- Engelleme sonrasi hedef, baglantiyi uygulama acisindan kullanilabilir ve anlasilir durumda tutmaktir.

```mermaid
flowchart LR
  A[Application] --> B[AXIS Policy Layer]
  B -->|policy check, allowed only| C[PostgreSQL]
  B -.->|blocked before database| A
```

## Karar Noktasi

- Policy check, PostgreSQL islemi calistirmadan once yapilir.
- AXIS sadece izin verilen islemleri veritabanina ulastirmayi hedefler.
- Belirsiz veya desteklenmeyen durumlarda varsayilan davranis sessiz gecirmek degil, kontrollu reddetmektir.
# Ana Tasarim Kararlari

## Varsayilan Davranis: Reddet

- AXIS, anlayamadigi veya desteklemedigi istegi sessizce gecirmez.
- Neden: Guvenlik katmaninda belirsizlik, izin anlamina gelmemelidir.
- Kanit: driver matrix ve pgwire testleri, desteklenmeyen batch ve COPY gibi akislarda fail-closed davranisi belgeliyor.

## Reddedilen Islem PostgreSQL'e Benzer Hata Uretir

- Engellenen isteklerde uygulama temiz bir PostgreSQL uyumlu hata gorur.
- Engellenen DELETE senaryolarinda SQLSTATE 42501 bekleniyor.
- Baglanti sonraki guvenli istekler icin kullanilabilir kalir.
- Neden: Uygulama tarafinda normal hata ve transaction kurtarma mantigi calisabilsin.

## Ham Degerleri Azaltan Policy Payload Tasarimi

- Parametreli Extended Query yolunda ham parametre degerleri policy payload'a konmaz; hash ozetleri ve sayisal/meta bilgiler kullanilir.
- psycopg3 ve asyncpg real-driver testleri, audit WAL icinde ham marker degerlerinin bulunmadigini kontrol eder.
- Prisma ve JDBC icin driver matrix, production leakage testinin henuz eklenmedigini acikca belirtiyor.
- Neden: Guvenlik karari icin gereken sinyal korunurken musteri verisinin policy ve log yuzeyine yayilmasi azaltılır.

## Savepoint Ile Kismi Kurtarma

- AXIS, savepoint kullanan transaction akıslarında riskli adimin durdurulmasi ve uygulamanin kendi kurtarma mantigiyla devam edebilmesi davranisini test ediyor.
- psycopg3, asyncpg, Prisma ve PostgreSQL JDBC icin savepoint recovery test matrisi kapsaminda.
- Neden: Kurumsal uygulamalar tum transaction'i kaybetmeden kontrollu kurtarma yapmak isteyebilir.

## Multi-statement Ve Session Kontrolleri

- Simple Query icinde birden fazla statement varsa her statement ayri siniflandirilir.
- Session parametrelerinde yalnizca dar ve gerekceli bir GUC whitelist'i gecirilir.
- search_path, role, session_authorization ve row_security reddedilir.
- Neden: Policy karari ile veritabaninin gercek nesne veya yetki baglami arasinda ayrisma olusmamali.
# Real-driver Kanitlari

## Test Sayisi Ve Kapsam

- docs/technical/driver_matrix.md, full real-driver suite icin 2026-07-05 tarihinde 37 testin gectigini belirtiyor.
- Kapsanan ekosistemler:
  - psycopg3, Python.
  - asyncpg, Python.
  - Prisma Client 6.19.3, TypeScript/JavaScript.
  - PostgreSQL JDBC, Java, pgjdbc 42.7.7.

## Ekosistem Bazinda Kanit Ozeti

- psycopg3 ile: parametreli sorgular, INSERT, UPDATE, engellenen DELETE, transaction abort, savepoint recovery, prepared reuse, SQL-level prepare/execute deny ve audit raw-value leakage kontrolleri dogrulandi.
- asyncpg ile: parametreli sorgular, prepared statement akisi, engellenen DELETE, transaction abort, savepoint recovery, SQL-level prepare/execute deny ve audit raw-value leakage kontrolleri dogrulandi.
- Prisma ile: parametreli SELECT, tekil INSERT, tekil UPDATE, engellenen DELETE, transaction abort ve interactive transaction icinde savepoint recovery dogrulandi.
- PostgreSQL JDBC ile: parametreli SELECT, tekil INSERT, tekil UPDATE, engellenen DELETE, transaction abort, savepoint recovery, prepared reuse ve default connection startup dogrulandi.

## Engelleme Ve Kurtarma Kanitlari

- Engellenen DELETE davranisi dort ekosistemde SQLSTATE 42501 beklentisiyle test ediliyor.
- Deny sonrasi baglanti kullanilabilirligi psycopg3, asyncpg, Prisma ve JDBC senaryolarinda kapsaniyor.
- JDBC batch execution MVP'de desteklenmiyor; test matrisi bu akisin temiz sekilde reddedildigini ve ayni baglantida sonraki kontrol sorgusunun calistigini belirtiyor.
- JDBC default connection artik assumeMinServerVersion=12 olmadan dogrulaniyor.

## Sızıntı Kontrolu Durumu

- psycopg3 ve asyncpg icin audit WAL marker kontrolleri var.
- Prisma ve JDBC icin production leakage testi henuz eklenmedi; bu durum driver matrix'te acikca not edilmis.
# Dürüst Sınırlar

Bu bolum bilincli olarak dahil edildi cunku net sinirlar belirsiz iddialardan daha guvenilirdir.

## Stored Function Ve Procedure Govdeleri Opaktır

- AXIS, client'in gonderdigi SQL metnini gorur.
- Onceden veritabaninda tanimlanmis function veya procedure govdesinin icini wire seviyesinde goremez.
- Bu AXIS'e ozgu bir eksik degil; PostgreSQL wire seviyesinde calisan benzer proxy'ler icin mimari sinirdir.
- Repo, bu davranisi regression test ile dogruluyor: function govdesinde veri silme olsa bile AXIS sadece function cagrisini policy'ler.
- Tamamlayici kontrol: PostgreSQL'in native GRANT ve REVOKE mekanizmalariyla function/procedure olusturma ve calistirma yetkileri kisitlanmalidir.

## COPY Protokolü Desteklenmiyor

- COPY FROM STDIN, COPY TO STDOUT ve CopyData alt-protokolu pgwire modunda desteklenmiyor.
- Beklenen davranis fail-closed: AXIS bulk data aktarimina girmeden reddeder.
- Repo testleri COPY FROM, COPY TO ve COPY reddi sonrasi baglanti kullanilabilirligi icin regression coverage iceriyor.

## JDBC Batch Ve Genel Pipelining Siniri

- pgjdbc batch execution MVP kapsaminda desteklenmiyor.
- Bunun nedeni ayni protocol cycle icinde birden fazla Bind/Execute akisinin ileri faz karari olarak ayrilmis olmasidir.
- Driver matrix bu davranisi desteklenmis gibi gostermiyor; temiz 42501 reddi ve baglanti kullanilabilirligi olarak belgeliyor.
- PIPELINING_SUPPORT=false iken JDBC batch yesil destek olarak isaretlenmemeli.

## Hassas Session Parametreleri Reddedilir

- search_path, role, session_authorization ve row_security reddedilir.
- Bunlari whitelist'e almak ayri bir guvenlik incelemesi ve bypass analizi gerektirir.
- search_path ozellikle hassastir; cunku unqualified tablo ve function isimlerinin PostgreSQL icinde nereye cozuldugunu degistirir.

## Deployment Sinirlari

- AXIS, kendisini tamamen bypass eden veritabani trafigini koruyamaz.
- Pilot veya uretim ortaminda ag, IAM, firewall, security group ve veritabani erisim kontrolleri AXIS'in zorunlu yol olmasini saglamalidir.
- Repo, full RBAC, SSO, TLS/mTLS deployment story, external KMS, external tamper-proof ledger veya multi-instance consensus'u bugunku kanitli uretim ozelligi olarak sunmuyor.
# Pilot Taslagi Ve Sonraki Adimlar

Bu bolum taslaktir, Selman ticari detaylari kendisi doldurmadan gonderilmemelidir.

## Baglayici Olmayan Pilot Secenekleri

- Tek uygulama ile sinirli pilot:
  - Kapsam dar tutulur.
  - Sadece belirlenen write path AXIS uzerinden gecirilir.
  - Basari kriterleri onceden yazilir.
- Gozlem modu ile baslama:
  - Ilk amac SQL sekillerini ve policy etkisini anlamaktir.
  - Engelleme politikalari daha sonra sinirli kapsamda acilir.
- Non-production ortamda teknik dogrulama:
  - Gercek driver ve ORM kombinasyonlari profillenir.
  - Migration, batch ve admin isleri kapsam disi veya ayri yol olarak tanimlanir.
- Dar policy pilotu:
  - Oncelik destructive write siniflarina verilir.
  - Bilinmeyen veya karmasik SQL sekilleri fail-closed olarak ele alinir.
- Ortak kanit paketi:
  - Test listesi, driver matrisi, limitation listesi ve pilot bulgulari birlikte saklanir.

## Gonderimden Once Doldurulacak Noktalar

- Pilot ortami: non-production, staging veya kontrollu production shadow.
- Kapsama alinacak uygulama ve veritabani rolu.
- Hangi islem siniflarinin engellenecegi veya sadece gozlemlenecegi.
- Basari kriterleri: baglanti uyumu, engelleme davranisi, rollback/kurtarma, audit gorunurlugu.
- Operasyonel sahiplik: kim policy degistirir, kim incident durumunda karar verir.
- Ticari ve hukuki sartlar: Selman tarafindan ayrica doldurulmali.
# Ek: Kisa Sozluk

- Extended Query: PostgreSQL client'larinin parametreli sorgulari parcalara ayirarak gonderdigi protocol akisi.
- Savepoint: Bir transaction icinde geri donulebilen ara nokta; tum islemi degil sadece sonraki riskli adimi geri almak icin kullanilabilir.
- Redaction/hash: Ham degeri saklamak yerine degeri gizlemek veya karsilastirilabilir bir ozetle temsil etmek.
- Fail-closed: Sistem emin degilse veya desteklemiyorsa istegi gecirmek yerine kontrollu sekilde reddetmesi.
- Tamper-evident audit: Repo testleri yerel hash-chain dogrulama ve imzali export kurcalama tespiti kapsiyor; harici degistirilemez ledger bugunku kanitli kapsam degil.
- Politika motoru: Istegin turunu, hedefini ve riskini degerlendirip izin, engelleme veya onay gerektirir sonucunu ureten karar katmani.
- PostgreSQL wire protocol: Uygulama ile PostgreSQL arasindaki dusuk seviyeli iletisim dili.
- COPY protokolu: PostgreSQL'in toplu veri alma veya verme yolu; AXIS pgwire modunda desteklenmez ve fail-closed reddedilir.
- GUC/session parameter: PostgreSQL oturum ayari; AXIS yalnizca guvenli kabul edilen dar bir listeyi gecirir.
- Driver matrix: Gercek client ve ORM'lerle hangi davranislarin test edildigini gosteren kanit tablosu.
# AXIS Documentation

Documentation is split by audience and purpose.

## Sections

- `technical/` - AXIS architecture, install/runbook, security model, runtime behavior, native PostgreSQL integration notes, and operator-facing technical docs.
- `demo/` - Pilot demo flow, real app integration notes, evidence package notes, benchmark guide, and demo limitations.
- `reviewer/` - External reviewer package, claim boundaries, attack checklist, feedback forms, and reviewer run/no-run guides.
- `presentation/` - Short presentation narrative and appendix.
- `explainers/` - Plain-language explanatory material.

`ALL_DOCS_COMBINED.md` is generated from the Markdown files under `docs/` and should not be edited by hand.

# AXIS Anti-Claims

This package is intentionally explicit about what it does not prove.

- This package does not claim native PostgreSQL wire compatibility.
- This package does not claim transparent drop-in proxy support.
- This package does not claim universal ORM support.
- This package does not claim production deployment readiness.
- This package does not claim application-layer routing alone prevents all direct DB bypass.
- This package does not claim live human approval while holding open database transactions.
- This package does not claim offline/disconnected corporate environment support.
- This package does not claim final compliance certification.
- This package does not claim enterprise-scale performance benchmarking.
- This package does not claim public open-source availability.

The current pilot demonstrates a source-visible integration pattern: FastAPI + SQLAlchemy routes protected writes through AXIS HTTP `/query`, while safe reads may go directly to PostgreSQL. That is useful evidence for architecture review, but it is not the same as a network-level PostgreSQL wire proxy.

If a reviewer sees a stronger claim implied by wording elsewhere in the package, treat this file as authoritative and flag the mismatch.
# AXIS Claims Matrix

This matrix maps the reviewer-package claims to concrete evidence and explicit limitations. It is intentionally narrow: it describes the current pilot package, not a future product surface.

| # | Claim | Evidence files | Verification command | Limitation | Reviewer question |
|---|---|---|---|---|---|
| 1 | AXIS routes protected ORM-generated writes through deterministic policy enforcement. | `demo/sample-business-app/backend/app/axis_session.py`, `demo/sample-business-app/backend/app/axis_client.py`, `demo/evidence/pilot-v1/responses/safe-write-response.json`, `demo/evidence/pilot-v1/audit/audit-sample-events.json` | `python scripts\verify_pilot_evidence.py` | Proven only for the pilot FastAPI + SQLAlchemy integration using the current HTTP `/query` adapter. Not native PostgreSQL wire compatibility. | Is the protected-write boundary clear enough for a real service team to maintain? |
| 2 | AXIS supports safe read/write split in the pilot integration. | `demo/evidence/pilot-v1/responses/safe-read-response.json`, `scripts/pilot_smoke_tests.py`, `demo/sample-business-app/backend/app/axis_session.py` | `python scripts\pilot_smoke_tests.py` | Safe reads may go directly to PostgreSQL by design. This does not prove direct write bypass prevention. | Is the split operationally realistic, or would teams route too much around AXIS? |
| 3 | AXIS returns structured security outcomes instead of generic backend failure. | `demo/evidence/pilot-v1/responses/approval-required-response.json`, `demo/evidence/pilot-v1/responses/blocked-operation-response.json`, `demo/evidence/pilot-v1/responses/approval-rejected-response.json` | `python scripts\verify_pilot_evidence.py` | Response shape is demonstrated through the sample backend, not every possible application framework. | Are the outcomes actionable enough for application code and operators? |
| 4 | AXIS supports approval-required flows using rollback plus explicit retry. | `demo/evidence/pilot-v1/responses/approval-required-response.json`, `demo/evidence/pilot-v1/responses/approval-retry-success-response.json`, `demo/evidence/pilot-v1/limitations/approval-retry-model.md` | `python scripts\run_pilot_demo.py` | Approval is not executed while holding the original database transaction open. The application must retry the exact operation with the approval id. | Is explicit retry the right tradeoff for safety and reliability? |
| 5 | AXIS avoids holding database transactions open while waiting for human approval. | `demo/evidence/pilot-v1/responses/transaction-approval-rollback-response.json`, `demo/evidence/pilot-v1/limitations/transaction-model.md`, `demo/sample-business-app/backend/app/main.py` | `python scripts\verify_pilot_evidence.py` | Demonstrated for pilot workflows. It does not prove every future transaction shape is safe. | Are rollback and retry semantics documented tightly enough for implementers? |
| 6 | AXIS blocks destructive operations according to policy. | `demo/sample-business-app/db/policies/axis_pilot_policy_v1.json`, `demo/evidence/pilot-v1/responses/blocked-operation-response.json`, `demo/evidence/pilot-v1/responses/transaction-blocked-rollback-response.json` | `python scripts\pilot_smoke_tests.py` | Blocks the policy-covered examples. It is not a claim that all possible destructive SQL forms are already covered. | Which bypass patterns should be tested before the next pilot? |
| 7 | AXIS generates audit evidence for protected operations. | `demo/evidence/pilot-v1/audit/audit-sample-events.json`, `demo/evidence/pilot-v1/audit/audit-verification-output.json`, `demo/evidence/pilot-v1/audit/audit-hash-chain-notes.md` | `python scripts\verify_pilot_evidence.py` | Current evidence uses AXIS audit output and endpoint verification. This package version does not include cryptographic signing for pre-generated evidence. | Is the audit evidence useful enough for incident review and governance? |
| 8 | AXIS pilot can be reproduced from a clean environment. | `demo/evidence/pilot-v1/clean-run/commands.md`, `demo/docker-compose.pilot.yml`, `scripts/capture_pilot_evidence.py` | `docker compose -f demo/docker-compose.pilot.yml down -v`; `docker compose -f demo/docker-compose.pilot.yml up -d --build`; `python scripts\pilot_smoke_tests.py` | Reproducibility depends on Docker, registries, local ports, and platform compatibility. Offline corporate environments are not supported. | What setup friction would block a reviewer or pilot team? |
| 9 | AXIS Control Plane/evidence visibility exists where current endpoints support it. | `demo/docker-compose.pilot.yml`, `docs/demo/PILOT_REVIEWER_DEMO_FLOW.md`, `demo/evidence/pilot-v1/audit/audit-verification-output.json` | Optional: `docker compose -f demo/docker-compose.pilot.yml --profile control-plane up -d` | The Control Plane is not the primary proof path in this package. Endpoint availability is limited to current implemented APIs. | What visibility is missing for an operator or AppSec reviewer? |
| 10 | AXIS current pilot uses HTTP adapter integration, not native wire compatibility. | `demo/sample-business-app/backend/app/axis_client.py`, `docs/reviewer/BYPASS_BOUNDARY_AND_DEPLOYMENT_ASSUMPTIONS.md`, `demo/evidence/pilot-v1/limitations/native-wire-gap.md` | Source inspection plus `python scripts\verify_pilot_evidence.py` | This is an explicit limitation. The package does not claim transparent drop-in PostgreSQL proxy behavior. | Is HTTP adapter validation sufficient before investing in native wire enforcement? |

Reviewer note: if a claim is not listed here, assume it is not claimed by this package.
# Bypass Boundary And Deployment Assumptions

The current pilot uses application-layer SQLAlchemy routing. Protected writes are routed through AXIS because the sample backend uses `AxisRoutingSession` and the AXIS HTTP `/query` adapter for protected write paths.

This is not the same as a network-level native PostgreSQL wire proxy.

## Current Integration Boundary

- Safe reads may go directly to PostgreSQL.
- Protected WRITE/DELETE/DDL paths are expected to go through AXIS by integration discipline.
- AXIS evaluates policy, returns structured outcomes, and emits audit evidence for protected operations.
- The HTTP adapter is the current integration model.

## What This Means

The pilot can demonstrate enforcement for code paths that use the integration correctly. It does not fully prove prevention of direct database write bypass by a misconfigured backend, leaked credential, or separate service with direct write access.

Direct DB write bypass prevention is not fully proven by this pilot. This is an explicit limitation, not a hidden claim.

## Production Hardening Would Require

- DB role separation.
- Credential control.
- Network policy.
- Restricted direct write credentials.
- Monitoring for direct database writes outside AXIS.
- Eventually native wire/proxy enforcement if transparent enforcement is required.

## Read/Write Split

The read/write split is intentional. It keeps low-risk read paths simple while routing protected mutations through AXIS. Reviewers should challenge whether this is operationally maintainable and what controls would be required to prevent accidental or intentional write bypass.
# Reviewer Docs

This section is for external review and claim-boundary material.

Start here:

- `AXIS_CLAIMS_MATRIX.md`
- `AXIS_ANTI_CLAIMS.md`
- `BYPASS_BOUNDARY_AND_DEPLOYMENT_ASSUMPTIONS.md`
- `REVIEWER_QUICKSTART.md`
- `REVIEWER_ATTACK_CHECKLIST.md`
- `REVIEWER_FEEDBACK_FORM.md`

# Review Package Modes

## A. Technical Source Review Package

- Includes source code.
- Intended for selected trusted technical reviewers.
- Goal: architecture tear-down.
- Recommended for AppSec, platform, SRE, and backend reviewers.

This is the first-round review mode.

## B. Executive / No-Run Review Package

- May rely on video, claims, pre-generated evidence, and docs.
- Source code optional.
- Intended for CISO/CTO/product-value preview.
- Goal: understand value, limitations, and adoption risk.

Do not confuse these two modes. First round is Technical Source Review Package. CISO/executive preview comes later.
# Reviewer Attack Checklist

Selected reviewers are invited to challenge AXIS deliberately. The goal of this package is technical damage discovery before broader executive or CISO outreach.

## Currently Covered By The Pilot

| Attack idea | Current evidence | Current status |
|---|---|---|
| Dangerous DELETE without WHERE | `demo/evidence/pilot-v1/responses/blocked-operation-response.json` | Covered for the pilot endpoint and policy rule. |
| ORM-generated risky update | `demo/evidence/pilot-v1/responses/approval-required-response.json` | Covered for role update requiring approval. |
| Approval replay attempt | `demo/evidence/pilot-v1/responses/approval-rejected-response.json` and approval retry flow | Partly covered through rejected approval retry behavior; broader replay cases need more tests. |
| Retry after rejected approval | `demo/evidence/pilot-v1/responses/approval-rejected-response.json` | Covered for the pilot flow. |
| Transaction rollback verification | `demo/evidence/pilot-v1/responses/transaction-approval-rollback-response.json`, `transaction-blocked-rollback-response.json` | Covered for two pilot workflows. |
| Audit evidence inspection | `demo/evidence/pilot-v1/audit/audit-sample-events.json` | Covered for captured protected operations. |

## Partially Covered

| Attack idea | What to inspect | Current status |
|---|---|---|
| Multi-statement SQL bypass attempt | Policy classifier and parser paths under `src/` plus pilot policy | Some parser hardening exists, but this package does not claim all multi-statement bypass classes are closed. |
| Dangerous UPDATE without WHERE | Policy rules and parser cases | Policy concept is present; the reviewer should test variants against current rules. |
| DDL attempt | Policy rules and destructive-operation handling | DDL blocking is in policy scope, but the pilot evidence focuses on selected business endpoints. |
| Malformed request handling | `src/gate/`, `src/errors.rs`, sample backend response mapping | Structured outcomes are shown; malformed-input coverage should be expanded by reviewers. |
| Policy metadata inspection | `demo/evidence/pilot-v1/audit/audit-sample-events.json` | Audit events include metadata; completeness should be reviewed. |

## Known Limitation

| Attack idea | Limitation |
|---|---|
| Direct DB bypass discussion | The pilot uses application-layer SQLAlchemy routing. Direct write bypass prevention requires DB role separation, credential control, and network policy. |
| Backend direct DB credential misuse discussion | The app still has direct PostgreSQL connectivity for safe reads. Preventing misuse is a deployment-hardening problem not fully proven here. |
| Missing/corrupt audit evidence scenario | The package shows audit generation and endpoint verification, but does not include independent offline recomputation for every event. |
| Policy mismatch scenario | The pilot does not prove every production policy drift or mismatch case is detected. |

## Future Work

- Native PostgreSQL wire/proxy enforcement.
- Stronger direct-write credential isolation.
- Expanded parser and multi-statement adversarial corpus.
- Broader ORM/framework coverage.
- Independent offline audit hash verification.
- Controlled benchmark work separate from this reviewer package.

Do not treat this checklist as a claim that all listed attack paths are solved.
# Reviewer Confidentiality Note

This package is shared for limited technical review.

It is not intended for production use. Do not redistribute without permission. Report security findings privately. Do not treat this as compliance certification.

This is not a public open-source release.

Run at your own risk. This package modifies local Docker state and may create local directories. VARUX assumes no liability for local environment issues. Use a disposable VM if restricted by corporate policy.
# Reviewer Delivery Note

Do not send only a blind ZIP to senior reviewers.

Do not attach ZIP directly to email. Corporate email gateways may silently quarantine ZIP attachments containing `.py`, `.sh`, `.yml`, or executable-looking files.

Recommended delivery:

1. Short personalized email.
2. Private 3-minute demo video link.
3. Hosted overview page or private document link.
4. Downloadable technical package via Google Drive, OneDrive, Tresorit, SendSafely, or similar secure file-sharing link.

Technical package should remain self-contained.

Hosted delivery may help understand whether reviewer opened material. Any analytics/tracking should be disclosed appropriately. Do not secretly track reviewers. Do not rely on analytics for technical correctness.
# Reviewer Feedback Form

Written feedback is optional. The primary feedback target is a 15-minute live teardown where you challenge the architecture directly.

Use 1 to 5 scores where 1 is weak/not credible and 5 is strong/credible.

## Reviewer Profile

- Name:
- Role:
- Profile: senior backend/platform engineer, AppSec/security architect, PostgreSQL/SRE/database engineer, product-minded technical advisor, or other:
- Relevant experience:
- Did you run the package locally: yes/no:
- Did you review source: yes/no:

## Scores

| Area | Score 1-5 | Notes |
|---|---:|---|
| Architecture credibility |  |  |
| Security value |  |  |
| Deployment realism |  |  |
| Evidence quality |  |  |
| Integration clarity |  |  |
| Enterprise pilot readiness |  |  |
| Urgency of the problem |  |  |
| Adoption likelihood |  |  |

## Technical Architecture Review

- What is the strongest architectural idea?
- What is the weakest architectural assumption?
- Where would this break in a real backend/platform environment?

## Integration Model Review

- Is HTTP `/query` integration credible as a pilot step?
- Is the SQLAlchemy routing boundary clear?
- What would be required before this could protect a production service?

## Transaction And Approval Review

- Is rollback plus explicit retry the right model?
- Are there replay, stale approval, or partial-write risks?
- What should be tested next?

## Audit/Evidence Review

- Are the response and audit samples useful?
- Is the evidence package easy to inspect?
- What evidence would you require before trusting the architecture?

## Security Concerns

- What is the highest-risk bypass path?
- What would an internal attacker or misconfigured service do first?
- What needs stronger proof?

## Operational Concerns

- What would fail during rollout?
- Which team would operate this?
- What would incident response need?

## Market Reality

1. If this architecture existed inside your company or infrastructure, what would be the biggest bureaucratic or technical blocker to implementation?
2. Is AXIS currently a vitamin, meaning nice to have, or a painkiller, meaning it addresses a critical operational/security pain?
3. Which team would own this product internally: security, platform, database, DevOps, governance, or another team?
4. What would need to be true before your organization would pilot AXIS?
5. What is the strongest reason not to adopt AXIS right now?

## Value Proposition

- What user or buyer pain is clearest?
- What value is still vague?
- What proof would make the value credible?

## Final Recommendation

Choose one:
- Not credible yet
- Technically interesting but not pilot-ready
- Pilot-worthy with limitations
- Strong technical direction, needs native wire path
- Strong candidate for narrow enterprise pilot

Additional comments:
# Reviewer No-Run Mode

Use no-run mode if you cannot or do not want to run untrusted code locally.

## What You Can Evaluate Without Running

- Architecture.
- Evidence shape.
- Claims and anti-claims.
- Source structure, if you received the source-visible package.
- Integration boundary.
- Risk model.
- Pre-generated evidence format.
- Bypass and deployment assumptions.

## What You Cannot Evaluate Without Running

- Local reproducibility.
- Local Docker compatibility.
- Local performance.
- Actual environment-specific behavior.
- Whether your own machine can build or run the stack.

No-run review can evaluate architecture and evidence shape, but it cannot independently verify local execution.

## Where To Inspect

- Start with `demo/pre-generated-evidence/README.md`.
- Inspect selected response examples under `demo/pre-generated-evidence/pilot-v1/selected-responses/`.
- Inspect selected audit samples under `demo/pre-generated-evidence/pilot-v1/selected-audit-events/`.
- Inspect verification examples under `demo/pre-generated-evidence/pilot-v1/verification-output/`.
- Read `docs/reviewer/AXIS_CLAIMS_MATRIX.md`.
- Read `docs/reviewer/AXIS_ANTI_CLAIMS.md`.
- Read `docs/reviewer/BYPASS_BOUNDARY_AND_DEPLOYMENT_ASSUMPTIONS.md`.
- Read `docs/reviewer/SOURCE_REVIEW_MODE.md` if reviewing source.

## Feedback

Preferred feedback is a 15-minute live walkthrough or teardown. Written feedback can use `docs/reviewer/REVIEWER_FEEDBACK_FORM.md`.
# Reviewer Outreach Email Template

## Version A: Senior Reviewer / CISO / Architect Short Email

Subject: Private technical teardown request for AXIS

Hi [NAME],

PERSONAL_REASON_FOR_THIS_REVIEWER

I am asking a small number of trusted reviewers to tear down AXIS privately before any CISO-level or public positioning.

AXIS is a deterministic control layer for protected PostgreSQL write paths. The current package shows a FastAPI + SQLAlchemy pilot where protected ORM writes go through policy enforcement, approval routing, rollback-safe retry, and audit evidence.

Demo video: VIDEO_LINK_PLACEHOLDER  
Hosted overview: HOSTED_OVERVIEW_LINK_PLACEHOLDER  
Package download: PACKAGE_DOWNLOAD_LINK_PLACEHOLDER

Could you give me a 15-minute live teardown? Written form is optional. Harsh feedback is preferred, especially on architecture, bypass paths, deployment realism, and whether this is worth piloting.

This is not a sales pitch in disguise. AXIS is in the engineering validation phase, not GTM. I am asking for architectural tear-down, not budget.

Turkish version: Bu gizli bir satış mesajı değildir. AXIS şu anda GTM/satış aşamasında değil, mühendislik doğrulama aşamasındadır. Bütçe değil, mimariyi acımasızca eleştirmenizi istiyorum.

## Version B: Technical Reviewer Email

Subject: Source-visible AXIS reviewer package - 15-minute teardown request

Hi [NAME],

PERSONAL_REASON_FOR_THIS_REVIEWER

AXIS is a deterministic control layer for protected PostgreSQL write paths.

This private reviewer package demonstrates AXIS protecting ORM-generated write operations in a realistic FastAPI + SQLAlchemy backend using policy enforcement, approval routing, rollback-safe retry, and audit evidence.

What is inside:
- Source-visible technical review package.
- Docker Compose pilot stack.
- Smoke tests and demo script.
- Evidence capture and verification.
- Pre-generated evidence for no-run review.
- Diagnostics collector if setup fails.
- Claims, anti-claims, bypass boundary, and attack checklist docs.

What is not claimed:
- Native PostgreSQL wire compatibility.
- Transparent enterprise drop-in proxy support.
- Universal ORM coverage.
- Production deployment readiness.
- Enterprise-scale benchmark results.

Start here: `demo/REVIEWER_START_HERE.md`

If setup fails:
- Windows: `python scripts\collect_reviewer_diagnostics.py`
- macOS/Linux: `python3 scripts/collect_reviewer_diagnostics.py`

I am asking for a 15-minute live teardown. Written form is optional. The feedback areas I care about most are architecture credibility, bypass paths, transaction and approval semantics, audit evidence, deployment realism, and whether this is pilot-worthy with current limitations.

This is not a sales pitch in disguise. AXIS is in the engineering validation phase, not GTM. I am asking for architectural tear-down, not budget.

Turkish version: Bu gizli bir satış mesajı değildir. AXIS şu anda GTM/satış aşamasında değil, mühendislik doğrulama aşamasındadır. Bütçe değil, mimariyi acımasızca eleştirmenizi istiyorum.
# AXIS Reviewer Quickstart

## Purpose

The AXIS reviewer demo verifies core security behavior on a clean machine with Docker:

- policy enforcement
- dangerous operation blocking
- approval routing
- audit evidence generation
- database state preservation after blocked operations

## Command

```sh
docker compose -f demo/docker-compose.reviewer.yml up --build
```

The reviewer client runs inside Docker. Do not install Python packages, npm packages, Rust tools, psql, or other local dependencies for this demo.

## Reviewer URLs

AXIS:

```text
http://localhost:65430
```

PostgreSQL:

```text
localhost:54320
```

Control Plane:

```text
Not included in demo/docker-compose.reviewer.yml. If a reviewer UI service is added later, it must use http://localhost:30000.
```

The reviewer compose file does not use host ports `5432`, `3000`, or `6543`.

## Evidence Output

Evidence files are written to:

```text
./demo-evidence-output
```

Expected files:

- `audit.log`
- `audit.wal`
- `approvals.sqlite`
- `reviewer-demo-report.json`
- `reviewer-demo-report.md`
- `evidence-bundle/audit-export.json`

## Success Condition

The demo is successful only if the `reviewer-client` container exits with code `0` and prints:

```text
AXIS REVIEWER DEMO: PASS
```

## Failure Condition

If any check fails, `reviewer-client` exits with code `1` and prints:

- failed check name
- expected result
- actual result
- short explanation

The JSON and Markdown reports in `./demo-evidence-output` contain the same failure details.
# Reviewer Rollout Plan

## Phase 1: Inner Technical Review

- Selected trusted reviewers.
- Source-visible.
- Live teardown preferred.
- Goal: find architectural flaws before CISO-level outreach.

Target profiles:

- Senior backend/platform engineer.
- AppSec/security architect.
- PostgreSQL/SRE/database engineer.
- Product-minded technical advisor.

First round objective is technical damage discovery.

## Phase 2: Executive / CISO Preview

- Cleaner no-run or guided package.
- Video-first.
- Evidence-first.
- Source optional.
- Goal: validate value perception and adoption blockers.

## Guardrails

- Do not send to CISO first.
- Do not ask for procurement.
- Do not position this as sales.
- Do not blur technical review with go-to-market outreach.
# Reviewer Setup And Teardown

## Start Stack

```bash
docker compose -f demo/docker-compose.pilot.yml up -d --build
```

## Stop Only

```bash
docker compose -f demo/docker-compose.pilot.yml down
```

## Clean Docker Volumes

```bash
docker compose -f demo/docker-compose.pilot.yml down -v
```

## Safe Cleanup

Windows:

```powershell
python scripts\clean_reviewer_environment.py
```

macOS/Linux:

```bash
python3 scripts/clean_reviewer_environment.py
```

## Dry Run

Windows:

```powershell
python scripts\clean_reviewer_environment.py --dry-run
```

macOS/Linux:

```bash
python3 scripts/clean_reviewer_environment.py --dry-run
```

## Non-Interactive Cleanup

Windows:

```powershell
python scripts\clean_reviewer_environment.py --yes
```

macOS/Linux:

```bash
python3 scripts/clean_reviewer_environment.py --yes
```

## What Is Removed

- Pilot Docker containers created by `demo/docker-compose.pilot.yml`.
- Pilot Docker volumes when `down -v` runs.
- Package-specific generated host artifacts such as `diagnostics/`, `demo-evidence-output/`, and local pilot runtime log folders when present and confirmed.

## What Is Intentionally Not Removed

- The source package itself.
- `dist/AXIS-External-Reviewer-Package-v1/`.
- Pre-generated evidence.
- Existing source files.
- Unrelated Docker images, containers, volumes, or networks.
- Global caches.
- Home directory files.
- Unrelated project folders.

The cleanup script is intentionally conservative. It removes AXIS reviewer-package artifacts only and does not clean unrelated Docker resources.

## Manual Inspection

To inspect remaining Docker resources:

```bash
docker compose -f demo/docker-compose.pilot.yml ps
docker volume ls
docker image ls
```

To inspect host artifacts, check:

- `diagnostics/`
- `demo-evidence-output/`
- `dist/AXIS-External-Reviewer-Package-v1/`
# Reviewer Troubleshooting

## Requirements

- Docker Desktop must be available.
- Docker Compose must work.
- Docker Hub/package registry access is required.
- Python 3.8+ is required.
- Disconnected/offline environments are not supported in this pilot package.

Corporate network warning:

This demo requires access to Docker Hub and package registries during build. If the reviewer is behind a corporate VPN, SSL inspection proxy, or restricted firewall, the build phase may fail before AXIS itself is running. Such failures should be treated as environment/setup failures, not product enforcement failures.

## Python Command Differences

- Windows: `python`
- macOS/Linux: `python3`

## Platform Check

Windows:

```powershell
python scripts\check_reviewer_platform.py
```

macOS/Linux:

```bash
python3 scripts/check_reviewer_platform.py
```

## Port Conflicts

The pilot expects these local ports:

- frontend: `8088`
- backend: `8000`
- AXIS: `6654`
- Postgres pilot port: `55432`

Stop conflicting local services or edit `demo/docker-compose.pilot.yml` only if you understand the impact on scripts and evidence.

## Docker Commands

Windows:

```powershell
docker compose -f demo/docker-compose.pilot.yml down -v
docker compose -f demo/docker-compose.pilot.yml up -d --build
docker compose -f demo/docker-compose.pilot.yml ps
```

macOS/Linux:

```bash
docker compose -f demo/docker-compose.pilot.yml down -v
docker compose -f demo/docker-compose.pilot.yml up -d --build
docker compose -f demo/docker-compose.pilot.yml ps
```

## Apple Silicon / ARM64

- This pilot should be checked on both `linux/amd64` and `linux/arm64` before broad external distribution.
- Apple Silicon has not been verified by this package unless the final delivery note explicitly says it was tested.
- Docker image architecture mismatch may cause build or runtime failures.
- Such failures should be separated from AXIS policy/enforcement failures.

## Setup Failed

If setup fails and you do not want to debug:

Windows:

```powershell
python scripts\collect_reviewer_diagnostics.py
```

macOS/Linux:

```bash
python3 scripts/collect_reviewer_diagnostics.py
```

Diagnostic output may include Docker logs and platform metadata. The diagnostic collector is designed not to include secrets, `.env` files, SSH keys, tokens, browser data, or unrelated Docker logs. Send the generated ZIP privately.
# Reviewer Video Demo Script

VIDEO_LINK_PLACEHOLDER: add private demo link here before external delivery.

Purpose: provide a precise 3-minute demo script for Loom, YouTube Unlisted, or private video recording.

Tone: practical, not salesy. No founder story. No hype. No native wire claim.

## Script

0:00 - 0:20

Say: "AXIS is a deterministic control layer for protected PostgreSQL write paths."

Show the package title and `demo/REVIEWER_START_HERE.md`.

0:20 - 0:45

Show reviewer package / Quickstart. Point to the four paths: 3-minute watch, no-run review, local run, setup failed.

0:45 - 1:20

Show clean pilot stack health:

```bash
docker compose -f demo/docker-compose.pilot.yml ps
python scripts\check_reviewer_platform.py
```

Open or show health evidence under `demo/evidence/pilot-v1/health/`.

1:20 - 1:50

Trigger safe write through ORM-backed business app:

```bash
python scripts\run_pilot_demo.py
```

Show "ORM customer insert routed through AXIS" and the returned audit event id.

1:50 - 2:20

Trigger risky operation and show `approval_required`. Show approval id only from the live response; do not invent one.

2:20 - 2:40

Show approval retry success after explicit approval resolution. Explain that the original DB transaction is not held open while waiting for approval.

2:40 - 3:00

Trigger blocked destructive operation and show audit evidence location:

- `demo/evidence/pilot-v1/responses/blocked-operation-response.json`
- `demo/evidence/pilot-v1/audit/audit-sample-events.json`

Close by asking for a 15-minute technical teardown.
# Source Review Mode

Architecture-level feedback requires source visibility. Black-box Docker behavior is not enough for AppSec, platform, or SRE review.

This source review package is intended only for selected trusted reviewers. It is not a public open-source release. Do not redistribute without permission.

Reviewers should focus on:

- Architecture.
- Enforcement boundaries.
- Bypass paths.
- Transaction handling.
- Audit evidence.
- Integration assumptions.
- Source/code organization.
- Policy decision flow.

Use `docs/reviewer/AXIS_CLAIMS_MATRIX.md` and `docs/reviewer/AXIS_ANTI_CLAIMS.md` as the source of truth for what is and is not claimed.
# Audit WAL Scaling And Derived Index

## Canonical Source Of Truth

The audit WAL remains the canonical source of evidence. The derived audit index is a read model only. It is safe to delete, rebuild, or ignore. `/audit/verify` verifies the WAL directly and does not use the index as proof.

Evidence Bundle V1 remains WAL-backed. `/audit/export` may use the index to choose candidate WAL offsets, but exported event bodies are re-read from WAL evidence before bundle hashing/signing.

## Audit Derived Index V1

AXIS writes the derived index to `AUDIT_INDEX_PATH`, defaulting to `./data/index/audit_index_v1.json`.

The persisted index has:

- metadata: `index_type=axis.audit_derived_index`, `index_version=1`, `built_at`, `source=audit_wal`, event counts, first/last IDs, last WAL hash, WAL byte length, WAL SHA-256, and `index_checksum`
- entries: `event_id`, `timestamp`, `event_type`, `decision`, `risk`, `actor`, `tenant`, `env`, `policy_id`, `approval_id`, `event_hash`, `previous_hash`, and `wal_offset`

The index intentionally does not store raw SQL, raw payloads, parameters, tokens, private keys, filesystem paths, or server environment values.

## Checksum Behavior

`index_checksum` is computed over deterministic canonical JSON with `metadata.index_checksum` removed. The checksum is therefore not self-referential.

The checksum protects the index payload from accidental or unauthorized edits. It does not make the index authoritative; WAL metadata and WAL-backed event loading still decide correctness.

## Build And Rebuild

Index rebuild is synchronous and derived from the WAL:

- verify WAL hash chain and event hashes first
- scan WAL records and collect safe searchable fields plus byte offsets
- compute WAL byte length and SHA-256
- compute index checksum excluding the checksum field
- persist the index atomically through a temporary file

Missing, corrupt, and stale indexes are rebuilt automatically when an audit read, export, or runtime metrics request needs the index.

## Missing, Stale, And Corrupt Indexes

Missing index:

- AXIS rebuilds from WAL when safe
- if rebuild cannot run safely, audit reads may fall back to strict WAL scan

Stale index:

- detected when WAL byte length, WAL SHA-256, or WAL last event hash differs from index metadata
- AXIS rebuilds from WAL when WAL verification passes

Corrupt index:

- detected by invalid JSON, invalid metadata, checksum mismatch, inconsistent event counts, or selected index entry mismatch against WAL
- AXIS discards the cached index state and rebuilds from WAL when safe

Malformed or hash-corrupt WAL:

- index is not marked ready
- AXIS does not silently repair the WAL
- `/audit/verify` continues to report WAL verification status directly
- audit APIs return structured WAL integrity errors instead of using index data as truth

## Pagination

Pagination cursors remain deterministic offset cursors: `v1:<matched_offset>`.

The index entries are stored in WAL order. Audit pages are produced newest-first by reversing that stable order, applying deterministic filters, and then applying the cursor offset. Repeated reads of the same index state produce the same next cursor.

Invalid cursors return a structured `invalid_pagination_cursor` error.

## Event Lookup

`GET /audit/events/:event_id` uses the index to find a candidate WAL offset by event ID or event hash. The returned full event is then loaded from the WAL at that offset. If the WAL event does not match the index entry, AXIS marks the index corrupt, rebuilds if safe, and otherwise falls back only when WAL reading remains safe.

`GET /audit/trace` is a read-only decision trace endpoint. It reconstructs traces from WAL-backed event bodies and does not treat runtime logs or index data as proof. The current implementation scans and verifies WAL evidence directly and keeps related events bounded by request `limit`.

## Filtering

`GET /audit/events` uses the index for candidate selection across actor, tenant, env, decision, risk, policy ID, approval ID, event type, and timestamp range. Returned summaries are loaded from WAL offsets and rechecked against the filters before response serialization.

## Export Selection

`GET /audit/export` uses the index only to select candidate event offsets. Evidence Bundle V1 events are built from WAL-loaded event details, not index-only bodies. Bundle `payload_sha256`, `event_hashes_sha256`, redaction flags, and optional Ed25519 signatures keep the existing V1 semantics.

If index/WAL candidate selection disagrees, WAL wins. AXIS marks the index corrupt, attempts a safe rebuild, or bypasses the index.

## Runtime Metrics

`GET /runtime/metrics` exposes safe index status:

```json
{
  "audit_index": {
    "status": "ready",
    "version": "1",
    "events_indexed": 123,
    "last_indexed_event_hash": "..."
  }
}
```

The status does not expose index paths, WAL paths, raw SQL, operator tokens, signing key material, or environment secrets.

## Operational Notes

- The index is disposable. Removing `AUDIT_INDEX_PATH` forces a rebuild on the next audit read/export/metrics call.
- WAL corruption remains a fail-fast startup condition through the audit logger.
- The active implementation keeps an in-memory cached index after safe validation and rebuilds when WAL bytes change.
- Hashing the WAL is still a linear byte read during validation/rebuild. This avoids trusting stale index metadata over WAL bytes, but it is not a substitute for future sealed-segment manifests.

## Known Limitations

- The index is a single JSON file. Very large deployments should move to sealed WAL segments plus segment indexes.
- Rebuild is synchronous in the request path today. A future operator job can rebuild asynchronously without changing WAL authority.
- The index does not provide retention, compaction, or archival semantics.
- `/audit/verify` remains a full WAL verification scan by design.
# AXIS Architecture Overview

## High-Level Purpose

AXIS is a deterministic control layer for PostgreSQL write paths. It sits between applications or operators and the database, classifies SQL, evaluates policy, enforces the decision, and records durable evidence.

The v0.6 package is designed for local technical review and pilot planning. It shows how the core enforcement, audit, approval, visibility, and policy lifecycle pieces fit together.

## Core Components

- Listener/API layer: Axum HTTP service exposing `/query`, approvals, audit, runtime, evidence, and policy lifecycle endpoints.
- SQL classification: PostgreSQL SQL parsing, normalization, fingerprinting, operation detection, target extraction, scope estimation, and risk signals.
- Session store: in-memory `session_id` scoped prepared statement metadata for AXIS-side `PREPARE`, `EXECUTE`, and `DEALLOCATE` enforcement.
- Policy engine: Versioned policy evaluation producing `ALLOW`, `BLOCK`, or `REQUIRE_APPROVAL`.
- Approval store: Local JSONL-backed pending and resolved approval records.
- Audit logger: WAL plus JSONL projection with event hashes and previous-hash linkage.
- Evidence verifier: Read-only hash-chain verification for audit evidence.
- Runtime log store: bounded in-memory operational log buffer exposed by `GET /logs`.
- Policy lifecycle store: Local immutable version files, active policy pointer, candidate state, validation, activation, and rollback.
- Runtime visibility endpoints: Health, runtime stats, runtime logs, audit explorer, policy status, and evidence verification.
- Control plane frontend: Next.js operator surface for dashboard, query console, approvals, audit, runtime, and policy lifecycle review.

## Request Decision Flow

```mermaid
flowchart TD
  A[Application or operator] --> B[AXIS POST /query]
  B --> C[Request validation]
  C --> S{Prepared command?}
  S -->|No| D[SQL classification]
  S -->|PREPARE| P[Classify inner SQL and register session metadata]
  S -->|EXECUTE| R[Resolve original SQL from SessionStore]
  S -->|DEALLOCATE| X[Remove session metadata]
  R --> D
  P --> J[Return response]
  X --> J
  D --> E[Policy evaluation]
  E --> F{Decision}
  F -->|ALLOW| G[Write audit evidence]
  G --> H[Execute through PostgreSQL path]
  H --> I[Record execution result evidence]
  I --> J[Return response]
  F -->|BLOCK| K[Write block evidence]
  K --> L[No database execution]
  L --> J
  F -->|REQUIRE_APPROVAL| M[Create approval record]
  M --> N[Write approval evidence]
  N --> O[No execution until approved]
  O --> J
```

Decision branches:

- `ALLOW`: AXIS writes decision evidence and executes through the configured PostgreSQL executor.
- `BLOCK`: AXIS writes block evidence and does not execute the SQL.
- `REQUIRE_APPROVAL`: AXIS creates an approval record, writes approval evidence, and does not execute until a later approval resolution.

Prepared statement branches:

- `PREPARE`: requires `session_id`, evaluates the inner SQL for risk context, registers AXIS-side metadata, writes audit evidence, and does not forward database-side `PREPARE`.
- `EXECUTE`: requires `session_id`, resolves the stored original SQL in that session, evaluates the original SQL through policy, and fails closed when unresolved.
- `DEALLOCATE` / `DEALLOCATE ALL`: require `session_id`, remove AXIS-side metadata, and write audit evidence.
- Allowed prepared `EXECUTE` is not blindly forwarded as raw PostgreSQL `EXECUTE` because HTTP sessions do not guarantee pooled backend connection affinity.

## Audit/Evidence Flow

Every important decision produces an audit event. AXIS records fields such as request identity, SQL fingerprint, operation, target, scope, risk signals, policy decision, final decision, reason code, matched rule, policy version, risk level, explanation, `previous_hash`, and `event_hash`.

Prepared statement decisions include `session_id` and prepared context fields when relevant: command, name, resolved flag, original SQL fingerprint, original operation/query type, and original risk level.

```mermaid
flowchart LR
  E1[Event 1] -->|event_hash becomes previous_hash| E2[Event 2]
  E2 -->|event_hash becomes previous_hash| E3[Event 3]
  E3 --> V[GET /evidence/verify]
  V --> R[Verification report]
```

Evidence behavior:

- New events include a `previous_hash` pointer to the last committed event hash.
- Each event has an `event_hash` calculated from canonical event content.
- On restart, AXIS reads the final non-empty audit record and continues the chain from that hash.
- `GET /evidence/verify` recomputes event hashes and verifies linkage without mutating the log.
- Malformed records, missing hash fields, or mismatched hashes are reported as invalid evidence. AXIS does not silently repair evidence.

## Runtime Logs Flow

Runtime logs are operational visibility, not audit evidence. AXIS keeps a bounded in-memory `VecDeque` of safe summaries and exposes it through `GET /logs`.

Runtime log entries include a stable runtime id, UTC timestamp, level, category, message, request id, actor/app/tenant context when available, decision, risk, classification summary, reason, matched rule, SQL fingerprint, policy metadata, approval id, and bounded metadata. They do not expose raw audit WAL records, raw headers, database credentials, operator tokens, backend URLs, or raw SQL bodies.

Current runtime log events include:

- AXIS runtime started.
- Policy loaded.
- Policy dry-run passed.
- Query decision emitted.
- Approval created.
- Approval resolved or rejected.
- Safe runtime errors such as execution uncertainty or evidence commit failure summaries.

The Control Plane reads runtime logs through `/api/axis/logs`. Real mode does not fabricate log rows; it shows either returned logs, a stable empty state, or a controlled unavailable/error state.

## Policy Lifecycle Flow

```mermaid
flowchart TD
  A[Active policy] --> B[Candidate JSON]
  B --> C[Validate]
  C -->|valid| D[Diff active vs candidate]
  D --> E[Dry-run representative SQL]
  E --> F[Create candidate version]
  F --> G[Activate with expected hash]
  G --> H[New active policy]
  H --> I[Rollback with expected hash]
  I --> A
  C -->|invalid| X[Reject candidate]
```

Lifecycle behavior:

- The active policy is loaded from the lifecycle store when present, or initialized from `POLICY_PATH`.
- Candidate policies are validated before storage.
- Diff compares candidate behavior against the active policy.
- Dry-run reuses classifier and policy evaluation without SQL execution, audit writes, or approval creation.
- Prepared `EXECUTE` dry-run has no durable session store attached and therefore fails safe as unresolved unless a future caller supplies an isolated session context.
- Candidate versions are stored as immutable local files with hash metadata.
- Activation requires the candidate ID and expected hash, validates again, then swaps the in-memory active policy.
- Rollback requires a valid stored version and expected hash.
- Old versions are retained for review and rollback.

## Control Plane

The Next.js control plane under `control-plane/` provides:

- Dashboard: health, runtime status, integrity, and recent activity.
- Query console: direct `/query` evaluation against the configured backend.
- Approval center: pending approvals and resolution.
- Evidence explorer: recent audit events and event detail.
- Policy lifecycle page: active policy, validation, diff, dry-run, versions, activation, and rollback.
- Runtime page: service status, audit posture, limits, and evidence verification.
- Logs page: real runtime logs from `/api/axis/logs`, preserving previous rows during background refresh.

The control plane should show real backend state in real mode. Mock mode is explicit server-side demo behavior only, not production evidence.

## Trust Boundaries

- Application to AXIS: SQL, request identity fields, and environment labels are untrusted inputs unless the deployment adds identity controls.
- AXIS to database: AXIS is trusted to enforce policy before forwarding allowed SQL to PostgreSQL.
- AXIS session id to PostgreSQL backend session: not trusted as equivalent in v0.8. Prepared state is AXIS-side security metadata, not a guarantee of database connection affinity.
- Operator to control plane: v0.6 has local settings and an optional operator token for mutating lifecycle endpoints; it is not full RBAC.
- Audit/evidence store: local files are trusted for local review but are not an external tamper-proof ledger.
- Runtime log store: in-memory operational visibility only; it is not trusted as durable proof.
- Policy store: local policy files and manifests are trusted local state; hash checks detect accidental or simple tampering but do not replace external key management or signed policy distribution.

## Design Constraints

- Deterministic over probabilistic: policy decides; AXIS does not guess with AI.
- Fail-safe defaults: unsupported or dangerous SQL shapes should not silently pass.
- No silent policy downgrade: policy activation and rollback require validation and expected-hash checks.
- No dry-run mutation: dry-run never executes SQL, writes audit evidence, or creates approvals.
- Prepared statement fail-safe: unresolved, cross-session, malformed, or missing-session `EXECUTE` never becomes silent `ALLOW`.
- Audit integrity over convenience: malformed or corrupted evidence is reported instead of hidden.
- Visibility without fake data: operator views should expose real state when connected to a live backend.
# Policy Manifest Startup Path

AXIS v0.9 makes the deployed policy manifest the authoritative startup input. `AXIS_POLICY_DIR` defaults to `./policies`, and `AXIS_POLICY_MANIFEST` defaults to `./policies/policy_manifest.json`. The manifest points to the active policy file, declares the expected policy version, and stores the raw policy file SHA-256.

Startup order is:

1. Open and verify audit WAL continuity.
2. Load and validate the policy manifest.
3. Resolve and hash the active policy file.
4. Parse and validate the policy model.
5. Run the deterministic activation dry-run corpus.
6. Write policy lifecycle audit events.
7. Start accepting traffic.

The in-memory runtime state stores `ActivePolicy`, which combines the existing policy model with `ActivePolicyMetadata` (`policy_id`, `policy_version`, `policy_sha256`, manifest path, policy path, and load time). The evaluator still uses the existing deterministic policy engine; the metadata is carried into decisions, audit records, health, and policy status responses.

Controlled reload is internal-only in v0.9 and disabled by default. No HTTP reload endpoint is introduced.
# AXIS Decision Traceability

## 1. Overview

AXIS decision traceability lets an operator inspect why AXIS made a recorded decision without re-executing SQL and without mutating security state.

The first endpoint is:

```text
GET /audit/trace
```

It reconstructs a bounded read-only trace from WAL-backed audit evidence and safe summaries.

## 2. What Decision Traceability Is

Decision traceability connects available evidence for a decision:

- request identity and `request_id`
- source audit event id/hash
- related audit events
- classifier output
- operation, target, scope, risk signals, and SQL fingerprint
- policy decision, final decision, reason code, matched rule, policy id/version/SHA-256
- approval id and approval resolution evidence when present
- execution state when present
- audit hash-chain position and verification status
- known limitations

## 3. What It Is Not

Decision traceability is not SQL replay, SQL re-execution, a database repair tool, an approval repair tool, or a policy mutation path.

It does not prove external business truth, database state outside AXIS, ticket correctness, or intent of an operator. It explains what AXIS recorded and can verify from its audit evidence.

## 4. Read-Only Guarantee

`GET /audit/trace` is read-only:

- it does not call the PostgreSQL executor
- it does not create approvals
- it does not resolve approvals
- it does not mutate policy state
- it does not append audit WAL events
- it does not write the JSONL projection
- it does not repair corrupted evidence

## 5. Source Of Truth Rules

The audit WAL is the source of truth. Runtime logs are operational visibility only. The derived audit index is disposable and is not proof.

The trace implementation scans and verifies WAL-backed audit events. It does not use runtime logs as evidence.

## 6. WAL-Backed Reconstruction

Trace reconstruction:

1. Validate that exactly one lookup key was provided.
2. Strictly scan the audit WAL.
3. Run existing hash-chain/event-hash verification.
4. Locate a source event.
5. Collect bounded related WAL events by matching `request_id`, `approval_id`, hash-chain neighborhood, and safe SQL fingerprint when available.
6. Build decision, classifier, policy, approval, execution, evidence, related-event, and limitation sections.

Unknown fields remain `null`, `unknown`, or empty arrays. AXIS does not fabricate missing classifier, policy, approval, or execution facts.

## 7. Derived Index Limitation

The derived audit index may be used by other audit APIs for candidate lookup. It is not authoritative evidence.

`GET /audit/trace` treats WAL records as authoritative and does not treat index contents as proof.

## 8. Runtime Logs Limitation

Runtime logs are not used to build decision traces. They can help operators find operational context, but they reset on restart, are bounded, and are not hash-chain evidence.

## 9. Endpoint Contract

```text
GET /audit/trace?event_hash=<sha256>
GET /audit/trace?event_id=<event_id>
GET /audit/trace?request_id=<request_id>
GET /audit/trace?approval_id=<approval_id>
```

Optional:

```text
limit=<1..100>
```

Default limit is `25`. The maximum is `100`.

Exactly one of `event_hash`, `event_id`, `request_id`, or `approval_id` is required.

## 10. Query Parameters

| Parameter | Meaning |
|---|---|
| `event_hash` | Locate a source event by 64-character SHA-256 audit event hash. |
| `event_id` | Locate a source event by audit event id. |
| `request_id` | Locate a decision trace for a request id. |
| `approval_id` | Locate approval-related decision evidence. |
| `limit` | Bound returned related events. Default `25`, maximum `100`. |

## 11. Response Example

```json
{
  "ok": true,
  "trace": {
    "trace_id": "trace_...",
    "source": {
      "lookup_type": "event_hash",
      "lookup_value": "...",
      "source_event_hash": "...",
      "source_event_id": "...",
      "request_id": "..."
    },
    "decision": {
      "decision": "BLOCK",
      "policy_decision": "BLOCK",
      "final_decision": "BLOCK",
      "reason_code": "policy_default_deny",
      "error_code": "policy_block",
      "risk": "CRITICAL"
    },
    "classification": {
      "operation": "DELETE",
      "query_type": "WRITE",
      "target": { "db": "prod_main", "schema": "public", "table": "orders" },
      "scope": "Batch",
      "risk_signals": ["delete_without_where"],
      "fingerprint": "..."
    },
    "policy": {
      "policy_id": "axis-prod-main",
      "policy_version": "prod_main@...",
      "policy_sha256": "...",
      "matched_rule": "default.fail_closed",
      "integrity": "recorded"
    },
    "approval": {
      "approval_id": null,
      "state": null,
      "resolved_by": null,
      "resolved_at": null,
      "decision": null
    },
    "execution": {
      "executed": false,
      "execution_state": "not_executed",
      "db_error_code": null
    },
    "evidence": {
      "event_hash": "...",
      "previous_hash": "...",
      "chain_position_known": true,
      "verification_status": "verified",
      "related_event_count": 3
    },
    "related_events": [],
    "limitations": [
      "Trace is reconstructed from WAL-backed audit evidence and safe summaries.",
      "Trace does not re-execute SQL.",
      "Trace does not prove external business truth."
    ]
  }
}
```

## 12. Error Behavior

Trace errors use the central AXIS structured error envelope.

| Condition | Code |
|---|---|
| Missing lookup key | `invalid_query_parameter` |
| Multiple lookup keys | `invalid_query_parameter` |
| Invalid limit | `invalid_query_parameter` |
| Malformed `event_hash` | `invalid_query_parameter` |
| Event/request not found | `audit_event_not_found` |
| Approval evidence not found | `approval_not_found` |
| Malformed WAL record | `audit_wal_corrupt` |
| Hash-chain/event-hash verification failure | `audit_verify_failed` |
| Unexpected internal failure | `internal_error` |

## 13. Security Redaction Rules

Trace responses do not include raw SQL, SQL parameters, operator tokens, DB URLs, backend URLs, private keys, env values, stack traces, raw WAL records, or private filesystem paths.

Trace includes SQL fingerprint and safe classifier/policy/approval summaries only.

## 14. Operator Use Cases

- Explain why a dangerous write was blocked.
- Confirm that an approval-required write did not execute at request time.
- Inspect rejection evidence after an approval was rejected.
- Distinguish DB timeout or execution-unknown evidence from ordinary policy block evidence.
- Link an audit event hash to related request, policy, approval, and execution records.

## 15. Known Limitations

- Trace reconstruction is only as complete as the recorded WAL evidence.
- Policy integrity in a trace means policy metadata was recorded in audit evidence; it does not revalidate historical policy bytes unless that evidence is separately verified.
- Related events are bounded by `limit`.
- Very large WAL files still require linear scan and verification.
- AXIS does not claim production readiness from this feature.

## 16. Test Commands

```powershell
cargo fmt --check
cargo check
cargo test trace_
curl.exe "http://localhost:6543/audit/trace?event_hash=<known_event_hash>"
curl.exe "http://localhost:6543/audit/trace?request_id=<known_request_id>"
curl.exe "http://localhost:6543/audit/trace?approval_id=<known_approval_id>"
cd control-plane
npm.cmd run typecheck
npm.cmd run lint
npm.cmd run build
npm.cmd run e2e:trace
```
# AXIS Error Code Registry

## 1. Overview

AXIS uses one structured public error contract for backend API failures and Control Plane display. The registry is implemented in `src/errors.rs` and is intended to keep API responses, runtime logs, runtime metrics, operator guidance, and docs aligned.

AXIS remains a hardened local technical review candidate. This registry does not make enterprise production-readiness claims.

## 2. Error Response Contract

```json
{
  "error": {
    "code": "audit_commit_failed",
    "message": "Audit evidence could not be committed before protected write execution.",
    "category": "audit",
    "severity": "security_critical",
    "request_id": "req_or_uuid_when_available",
    "safe_to_retry": false,
    "operator_action": "Keep protected write traffic disabled. Check audit WAL availability, disk space, and write permissions. Run audit verification before restoring protected write traffic.",
    "details": {
      "reason": "safe public reason only"
    }
  }
}
```

`request_id` and `details` are optional. `safe_to_retry`, `operator_action`, `message`, `category`, `severity`, and `code` are always present in structured errors.

## 2.1 QueryResponse Embedded Error Metadata

The query API has two compatible structured-error forms:

- Request-level failures that cannot safely return a query decision, such as invalid JSON, oversized bodies, oversized SQL, request timeout, rate limit, and classifier/parser rejection, may return the standalone `{ "error": ... }` envelope.
- Enforcement decisions that already use the stable `QueryResponse` contract keep top-level fields such as `decision`, `policy_decision`, `reason_code`, `error_code`, `request_id`, `approval_id`, policy metadata, and audit behavior. When such a response is fail-closed or non-executed, AXIS also includes an optional `error` object with the same `AxisErrorBody` shape.

Example embedded query error:

```json
{
  "decision": "BLOCK",
  "policy_decision": "BLOCK",
  "reason_code": "policy_default_deny",
  "error_code": "policy_block",
  "error": {
    "code": "policy_block",
    "message": "Request was blocked by policy.",
    "category": "policy",
    "severity": "warning",
    "request_id": "76cf9f10-50f3-4b29-a536-8e0221501e71",
    "safe_to_retry": false,
    "operator_action": "Do not bypass AXIS. Review matched policy rule and audit evidence before changing policy.",
    "details": {
      "reason": "policy_default_deny",
      "matched_rule": "default.fail_closed",
      "decision": "BLOCK",
      "policy_decision": "BLOCK",
      "policy_id": "prod-main",
      "policy_version": "prod-main-v1",
      "sql_fingerprint": "..."
    }
  }
}
```

Clients should prefer `error` when present and retain `error_code` as a backwards-compatible fallback. The embedded object is generated from `AxisErrorCode`; listener code must not duplicate public messages or operator actions.

## 3. Category Model

Categories are: `request`, `sql`, `policy`, `approval`, `audit`, `audit_index`, `database`, `auth`, `runtime`, `config`, `evidence`, and `internal`.

## 4. Severity Model

Severities are: `info`, `warning`, `error`, `critical`, and `security_critical`.

`security_critical` is reserved for failures that can affect write-path safety, evidence integrity, approval execution safety, or policy lifecycle integrity.

## 5. HTTP Status Mapping

The canonical mapping lives in `src/errors.rs` on `AxisErrorCode::status()`.

| HTTP | Codes |
|---|---|
| 400 | `invalid_json`, `invalid_request_schema`, `invalid_query_parameter`, `missing_required_field`, `empty_sql`, `parser_error`, `parser_unsupported_syntax`, `unsupported_sql_shape`, `multi_statement_rejected`, `prepared_statement_requires_session`, `unresolved_prepared_statement`, `duplicate_prepared_statement`, `unsupported_prepared_statement`, `unsafe_read_shape`, `invalid_pagination_cursor` |
| 401 | `operator_auth_required` |
| 403 | `operator_auth_invalid`, `operator_auth_weak_config`, `policy_block`, `approval_rejected`, `approval_execution_blocked`, `evidence_signature_required` |
| 404 | `not_found`, `approval_not_found`, `audit_event_not_found` |
| 409 | `approval_already_resolved`, `policy_version_conflict`, `policy_activation_failed`, `policy_rollback_failed` |
| 413 | `request_body_too_large`, `sql_too_large` |
| 422 | `policy_validation_failed`, `policy_dry_run_failed`, `policy_manifest_missing`, `policy_manifest_invalid`, `policy_checksum_mismatch`, `policy_version_mismatch`, `evidence_bundle_invalid`, `evidence_signature_invalid` |
| 429 | `rate_limited` |
| 500 | `internal_error`, `approval_resolution_failed`, `evidence_export_failed`, `db_execution_failed`, `audit_verify_failed`, `config_invalid` |
| 503 | `db_unavailable`, `db_pool_exhausted`, `audit_unavailable`, `audit_commit_failed`, `audit_wal_corrupt`, `audit_index_corrupt`, `audit_index_rebuild_failed`, `approval_store_corrupt`, `runtime_unhealthy`, `policy_not_loaded`, `approval_audit_failed` |
| 504 | `request_timeout`, `db_timeout`, `execution_state_unknown` |

## 6. Full Error Code Table

| Code | Meaning | HTTP | Category | Severity | Retry | Reported to | Operator action |
|---|---|---:|---|---|---|---|---|
| `invalid_json` | Request body is not valid JSON. | 400 | request | error | yes | runtime_log, metrics | Correct client JSON. |
| `invalid_request_schema` | Request failed AXIS schema validation. | 400 | request | error | yes | runtime_log, metrics | Correct request fields. |
| `request_body_too_large` | Body exceeds configured limit. | 413 | request | warning | no | runtime_log, metrics | Reduce request size. |
| `sql_too_large` | SQL exceeds configured limit. | 413 | sql | warning | no | runtime_log, metrics, audit_for_query_rejection | Reduce SQL payload. |
| `invalid_pagination_cursor` | Cursor cannot be decoded. | 400 | request | error | yes | runtime_log, metrics | Refresh list cursor. |
| `invalid_query_parameter` | Query parameter is invalid. | 400 | request | error | yes | runtime_log, metrics | Correct query parameter. |
| `missing_required_field` | Required field is missing or empty. | 400 | request | error | yes | runtime_log, metrics | Send required field. |
| `empty_sql` | SQL is empty. | 400 | sql | error | yes | runtime_log, metrics, audit_for_query_rejection | Submit classifiable SQL. |
| `multi_statement_rejected` | More than one statement was submitted. | 400 | sql | warning | no | runtime_log, metrics, audit_for_query_rejection | Submit one statement. |
| `unsupported_sql_shape` | SQL shape is unsupported. | 400 | sql | error | yes | runtime_log, metrics, audit_for_query_rejection | Review SQL shape. |
| `parser_error` | SQL parser failed safely. | 400 | sql | error | yes | runtime_log, metrics, audit_for_query_rejection | Correct SQL syntax. |
| `parser_unsupported_syntax` | SQL syntax is unsupported by the AXIS parser. | 400 | sql | error | yes | runtime_log, metrics, audit_for_query_rejection | Treat as a parser coverage gap; add parser support before allowing. |
| `prepared_statement_requires_session` | PREPARE/EXECUTE/DEALLOCATE lacks `session_id`. | 400 | sql | error | yes | runtime_log, metrics, audit_for_query_rejection | Provide scoped session id. |
| `unresolved_prepared_statement` | EXECUTE cannot resolve statement in session. | 400 | sql | error | yes | runtime_log, metrics, audit_for_query_rejection | Refresh session state. |
| `duplicate_prepared_statement` | Statement name already exists in session. | 400 | sql | error | yes | runtime_log, metrics, audit_for_query_rejection | Deallocate or use another name. |
| `unsupported_prepared_statement` | Prepared command/name is unsupported. | 400 | sql | error | yes | runtime_log, metrics, audit_for_query_rejection | Use supported prepared lifecycle. |
| `unsafe_read_shape` | Read-like SQL has write-capable behavior. | 400 | sql | warning | no | runtime_log, metrics, audit_for_query_rejection | Treat as unsafe; rewrite SQL. |
| `policy_not_loaded` | No validated policy is loaded. | 503 | policy | critical | no | runtime_log, metrics, audit_if_startup_or_lifecycle | Disable write traffic until loaded. |
| `policy_manifest_missing` | Policy manifest is missing. | 422 | policy | error | no | runtime_log, metrics, audit_policy_lifecycle | Restore manifest. |
| `policy_manifest_invalid` | Manifest/schema/path is invalid. | 422 | policy | error | no | runtime_log, metrics, audit_policy_lifecycle | Verify manifest and policy bytes. |
| `policy_checksum_mismatch` | Manifest checksum does not match policy. | 422 | policy | security_critical | no | runtime_log, metrics, audit_policy_lifecycle | Do not serve this policy; verify SHA-256. |
| `policy_version_mismatch` | Manifest and policy versions disagree. | 422 | policy | error | no | runtime_log, metrics, audit_policy_lifecycle | Fix version metadata. |
| `policy_validation_failed` | Candidate policy failed validation. | 422 | policy | error | no | runtime_log, metrics | Fix validation errors. |
| `policy_dry_run_failed` | Policy activation dry-run failed. | 422 | policy | error | no | runtime_log, metrics, audit_policy_lifecycle | Fix unsafe dry-run findings. |
| `policy_activation_failed` | Activation failed. | 409 | policy | security_critical | no | runtime_log, metrics, audit_policy_lifecycle | Refresh lifecycle state and verify hashes. |
| `policy_rollback_failed` | Rollback failed. | 409 | policy | security_critical | no | runtime_log, metrics, audit_policy_lifecycle | Verify rollback target and hashes. |
| `policy_version_conflict` | Version status conflicts with operation. | 409 | policy | error | no | runtime_log, metrics | Refresh lifecycle state. |
| `policy_block` | Policy blocked request. | 403 | policy | warning | no | runtime_log, metrics, audit | Do not bypass; review matched rule. |
| `approval_not_found` | Approval id not found. | 404 | approval | info | yes | runtime_log, metrics | Refresh approval state. |
| `approval_already_resolved` | Approval was already resolved. | 409 | approval | error | yes | runtime_log, metrics, audit_if_resolution_attempt | Verify final decision in audit. |
| `approval_rejected` | Approval was rejected. | 403 | approval | warning | no | runtime_log, metrics, audit | Do not execute rejected request. |
| `approval_store_corrupt` | Approval store integrity failed. | 503 | approval | critical | no | runtime_log, metrics | Stop approval mutation; preserve store. |
| `approval_resolution_failed` | Approval resolution failed unexpectedly. | 500 | approval | error | no | runtime_log, metrics | Check runtime logs with request id. |
| `approval_execution_blocked` | Approved execution blocked by safety rule. | 403 | approval | security_critical | no | runtime_log, metrics, audit | Do not manually replay. |
| `approval_audit_failed` | Approval audit evidence failed. | 503 | approval | security_critical | no | runtime_log, metrics, audit | Restore WAL health before approvals. |
| `audit_commit_failed` | Audit evidence could not be committed. | 503 | audit | security_critical | no | runtime_log, metrics, audit_integrity_state | Disable protected writes; verify WAL. |
| `audit_unavailable` | Audit evidence cannot be read or written. | 503 | audit | critical | no | runtime_log, metrics | Restore WAL availability. |
| `audit_wal_corrupt` | WAL failed integrity checks. | 503 | audit | security_critical | no | runtime_log, metrics | Preserve WAL; run recovery runbook. |
| `audit_verify_failed` | Audit verification failed. | 500 | audit | critical | no | runtime_log, metrics | Verify files and recovery process. |
| `audit_index_corrupt` | Derived audit index is corrupt. | 503 | audit_index | critical | no | runtime_log, metrics | Treat index as disposable; verify WAL. |
| `audit_index_rebuild_failed` | Derived index rebuild failed. | 503 | audit_index | critical | no | runtime_log, metrics | Use WAL fallback after verification. |
| `audit_event_not_found` | Event id/hash not found. | 404 | audit | info | yes | runtime_log, metrics | Verify event identifier. |
| `evidence_bundle_invalid` | Evidence bundle is invalid. | 422 | evidence | error | no | runtime_log, metrics | Regenerate from verified WAL. |
| `evidence_export_failed` | Evidence export failed. | 500 | evidence | error | no | runtime_log, metrics | Verify WAL and filters. |
| `evidence_signature_invalid` | Evidence signature/config is invalid. | 422 | evidence | error | no | runtime_log, metrics | Verify signing public/private key config. |
| `evidence_signature_required` | Signature is required but absent. | 403 | evidence | error | no | runtime_log, metrics | Enable valid signing configuration. |
| `db_unavailable` | Database is unavailable. | 503 | database | critical | no | runtime_log, metrics | Check database connectivity. |
| `db_pool_exhausted` | DB pool could not provide connection. | 503 | database | critical | no | runtime_log, metrics, audit_for_write_path_failure | Reduce load or increase pool after review. |
| `db_timeout` | DB operation timed out. | 504 | database | security_critical | no | runtime_log, metrics, audit_execution_unknown | Treat execution state as unknown. |
| `db_execution_failed` | DB failed before confirmation. | 500 | database | error | no | runtime_log, metrics, audit_execution_failed | Confirm not forwarded before retry. |
| `execution_state_unknown` | Execution result is unknown. | 504 | database | security_critical | no | runtime_log, metrics, audit_execution_unknown | Reconcile database state before retry. |
| `operator_auth_required` | Operator auth is missing. | 401 | auth | error | yes | runtime_log, metrics | Provide configured auth path. |
| `operator_auth_invalid` | Operator auth is invalid. | 403 | auth | warning | yes | runtime_log, metrics, audit_for_sensitive_mutation | Verify token source; do not expose token. |
| `operator_auth_weak_config` | Operator auth config is weak. | 403 | auth | warning | no | runtime_log, metrics | Replace weak token config. |
| `request_timeout` | AXIS request timed out. | 504 | runtime | error | no | runtime_log, metrics | Check saturation and retry idempotent calls only. |
| `rate_limited` | Request rate exceeded. | 429 | runtime | warning | yes | runtime_log, metrics | Reduce rate or review limit config. |
| `config_invalid` | Runtime config is invalid. | 500 | config | error | no | runtime_log | Fix config without logging env values. |
| `runtime_unhealthy` | Runtime health is degraded. | 503 | runtime | critical | no | runtime_log, metrics | Check health, logs, WAL, and DB. |
| `not_found` | Resource or route not found. | 404 | runtime | info | yes | runtime_log, metrics | Verify identifier or endpoint. |
| `internal_error` | Internal fallback error. | 500 | internal | error | no | runtime_log, metrics | Capture request_id and sanitized logs. |

## 7. Reporting Destination Rules

Runtime logs receive structured summaries for meaningful API errors emitted through the central helpers, including embedded query-response errors. Runtime metrics count errors by code, category, and severity where the emitting path has `AppState`.

Audit evidence is intentionally narrower than runtime logs. AXIS writes audit evidence for security-relevant write-path and policy/approval lifecycle events, not for every client formatting problem.

## 8. Runtime Log Behavior

Runtime log entries may include timestamp, level, category, severity, `error_code`, `request_id`, endpoint, method, safe message, and operator action. Logs must not include raw SQL, stack traces, headers, secrets, private paths, DB URLs, or raw WAL records.

## 9. Audit Evidence Behavior

Audit-relevant examples include `policy_block`, `approval_rejected`, `audit_commit_failed`, `audit_wal_corrupt`, `policy_checksum_mismatch`, `policy_activation_failed`, `policy_rollback_failed`, `execution_state_unknown`, `db_timeout`, `approval_audit_failed`, and `approval_execution_blocked`.

Invalid JSON, ordinary bad query parameters, and other request-shape failures are runtime visibility only unless they pass through an existing security-relevant query rejection path.

`GET /audit/trace` uses the same structured envelope for trace lookup and WAL integrity failures. Missing or multiple lookup keys, malformed event hashes, and invalid limits return `invalid_query_parameter`. Missing event/request evidence returns `audit_event_not_found`; missing approval evidence returns `approval_not_found`. Malformed WAL records return `audit_wal_corrupt`; hash-chain or event-hash verification failures return `audit_verify_failed`.

## 10. Runtime Metrics Behavior

`/runtime/metrics` includes `errors.total`, `errors.by_code`, `errors.by_category`, `errors.by_severity`, `errors.last_error_at`, and `errors.last_security_critical_error_at`.

Known limitation: legacy helper paths that do not receive `AppState` can return structured errors without incrementing counters until those helpers are state-aware.

## 11. Operator Action Guide

Operator action text is defined per error code in `AxisErrorCode::operator_action()`. High-impact actions intentionally instruct operators to preserve WAL, disable protected write traffic when evidence is unsafe, avoid blind retries, and keep credentials out of logs and browsers.

## 12. Retry Guidance

`safe_to_retry=true` means the request can usually be retried after correcting client state, credentials, cursor, or rate. It does not mean AXIS will automatically retry protected writes.

`db_timeout`, `execution_state_unknown`, `audit_commit_failed`, approval audit failures, and policy lifecycle integrity failures are not safe to retry automatically.

## 13. Security Redaction Rules

Public error details may include only bounded safe fields such as `reason`, `limit_bytes`, `timeout_ms`, `retry_after_seconds`, `endpoint`, `method`, `policy_id`, `policy_version`, `policy_sha256`, `matched_rule`, `decision`, `policy_decision`, `risk`, `operation`, `query_type`, `scope`, `sql_fingerprint`, `session_id`, `prepared_statement_name`, `prepared_statement_command`, `request_id`, `approval_id`, `event_hash`, `cursor_format`, `safe_state`, `integrity_state`, and `execution_state`.

Public errors and Control Plane error views must not expose raw SQL, normalized SQL that reveals content, full request bodies, auth headers, tokens, env values, private paths, DB connection strings, stack traces, raw WAL records, or unbounded user-controlled strings.

## 14. Frontend Display Rules

The Control Plane validates the structured error body, displays code/message/category/severity/request_id/safe_to_retry/operator_action/safe details, and sanitizes details again client-side before rendering. It must not render backend URLs, tokens, raw SQL, stack traces, or private paths from error details.

## 15. Examples

Invalid JSON:

```json
{
  "error": {
    "code": "invalid_json",
    "message": "Request body must be valid JSON.",
    "category": "request",
    "severity": "error",
    "safe_to_retry": true,
    "operator_action": "Correct the client request shape. Do not include secrets or raw SQL in troubleshooting tickets."
  }
}
```

Database timeout:

```json
{
  "error": {
    "code": "db_timeout",
    "message": "Database operation exceeded configured timeout.",
    "category": "database",
    "severity": "security_critical",
    "request_id": "76cf9f10-50f3-4b29-a536-8e0221501e71",
    "safe_to_retry": false,
    "operator_action": "Check database health and query latency. Treat execution state as unknown when reported. Do not automatically retry protected writes.",
    "details": {
      "timeout_ms": 8000,
      "execution_state": "unknown"
    }
  }
}
```

## 16. Known Limitations

Some query decision failures still return the established query decision body with stable `reason_code`/`error_code` instead of the full structured error envelope to preserve ALLOW / BLOCK / REQUIRE_APPROVAL behavior and existing regression expectations.

Some policy, audit, and evidence helper paths produce structured errors without request IDs because no request context exists at that layer.

Runtime metrics count central state-aware error emissions. A future pass should move all helper-level errors to state-aware helpers to eliminate metric gaps and possible double counts where a runtime error log and structured response are emitted separately.
# AXIS External Review Summary

## What AXIS Is

AXIS is a deterministic database write-path control layer for PostgreSQL. It sits between applications or operators and the database, classifies SQL, evaluates policy, enforces `ALLOW`, `BLOCK`, or `REQUIRE_APPROVAL`, and records audit evidence.

The v0.6 package is meant for serious external review and early pilot evaluation. It is a local technical core with operator visibility and policy lifecycle controls, not a SaaS product.

## Why It Exists

Production databases can be damaged by writes that pass through legitimate channels: application bugs, internal tools, scripts, migrations, compromised credentials, and operator mistakes.

Permissions and logs are necessary but often insufficient. AXIS adds a deterministic control point directly before PostgreSQL receives the operation. The aim is to make risky writes explainable, enforceable, reviewable, and auditable before execution.

## What Is Implemented Today

- Rust backend service with Axum HTTP API.
- SQL validation and PostgreSQL-focused classification.
- Policy engine with `ALLOW`, `BLOCK`, and `REQUIRE_APPROVAL`.
- Approval workflow with local JSONL store.
- Audit evidence generation with hash-chain fields.
- Evidence verification endpoint.
- Read-only decision traceability endpoint and Control Plane trace drawer for WAL-backed explanation of recorded decisions.
- Runtime visibility endpoints.
- Central structured error taxonomy with operator actions and safe Control Plane rendering.
- Fail-closed query decision responses embed central structured error metadata while preserving the existing query response contract.
- Next.js control plane for operators.
- Policy validation, diff, dry-run, candidate versions, activation, and rollback.
- Optional operator token for mutating policy lifecycle endpoints.
- Docker Compose local PostgreSQL path.
- Backend regression, parser, approval, audit continuity, and policy lifecycle tests.

## What Is Not Implemented Yet

- Full RBAC or SSO.
- TLS/mTLS deployment story.
- Distributed consensus or multi-instance coordination.
- External KMS or signed policy distribution.
- Tamper-proof external audit ledger.
- Formal compliance mapping or certification.
- SaaS control plane.
- Dedicated operator audit stream.
- Production-grade log retention, archival, and rotation policy.

## What Should Be Reviewed

- SQL classifier behavior against realistic application and migration SQL.
- Fail-safe behavior for unsupported SQL shapes.
- Policy defaults and rule semantics.
- Approval workflow integrity.
- Audit evidence fields and hash-chain verification.
- Decision traceability behavior, including read-only guarantees and redaction boundaries.
- Policy lifecycle controls and rollback behavior.
- Control plane accuracy in real backend mode.
- Local file-backed store assumptions.
- Deployment controls required before production use.

## How To Run It

Backend checks:

```cmd
cargo fmt --check
cargo check
cargo test
```

Local stack:

```cmd
docker compose up --build
```

Control plane:

```cmd
cd control-plane
npm install
npm run build
npm run dev
```

Open:

```text
Backend: http://localhost:6543
Control plane: http://localhost:3000
```

API demo:

```cmd
python scripts/demo_axis_flow.py
```

Detailed docs:

- [Install Guide](AXIS_INSTALL_GUIDE.md)
- [Demo Flow](../demo/AXIS_DEMO_FLOW.md)
- [Architecture Overview](AXIS_ARCHITECTURE_OVERVIEW.md)
- [Security Model](AXIS_SECURITY_MODEL.md)
- [Error Code Registry](AXIS_ERROR_CODE_REGISTRY.md)
- [Threat Model](AXIS_THREAT_MODEL.md)
- [Pilot Checklist](../demo/AXIS_PILOT_CHECKLIST.md)

## What A Successful Review Looks Like

A successful review should show that:

- The reviewer can run AXIS locally.
- Safe reads behave as expected.
- Risky writes are blocked or routed to approval according to policy.
- Approval-required writes do not execute before approval.
- Audit events are produced.
- Evidence verification is understandable and reproducible.
- Policy dry-run predicts decision behavior without mutation.
- Candidate activation and rollback are controlled by validation and expected hashes.
- Known limitations are clear and not hidden.

## Open Technical Questions

- Which SQL forms from real applications need classifier expansion?
- What identity model should bind request fields to verified principals?
- What transport security and network placement are required for pilot environments?
- Should audit evidence be exported to an external append-only store?
- What policy signing or KMS-backed verification is required?
- How should multi-instance deployments coordinate policy and audit continuity?
- What operator actions require a dedicated audit stream?
- What latency budget is acceptable for write-path enforcement?
# AXIS Failure Mode Matrix

AXIS v1 is not a production-ready enterprise product. It is a local hardened v1 security core. This matrix documents the tested local failure-mode baseline and the expected safety property for each case.

## Matrix

| Failure mode | Test source | Expected behavior | Verified result | Remaining risk |
| --- | --- | --- | --- | --- |
| PostgreSQL down | `axis_chaos_test.py::postgres_down_fail_safe` | Risky writes must not receive unsafe `200 ALLOW`; service recovers after PostgreSQL restart. | PASS | Availability is degraded while DB is down; no external alerting. |
| Audit path unwritable | `axis_chaos_test.py::audit_unwritable_no_write_execution` | Writes must not execute when audit evidence cannot be written. | PASS | No automated remediation or disk-space monitoring. |
| Invalid policy JSON | `axis_chaos_test.py::invalid_policy_startup_fail_fast` | AXIS must fail startup or become unhealthy instead of running with invalid policy. | PASS | No signed policy or centralized policy distribution. |
| Corrupt approval store | `axis_chaos_test.py::approval_store_corrupt_fail_safe` | AXIS must fail safely; corrupt approval state must not allow writes. | PASS | Recovery is manual; no approval log compaction or repair tool. |
| Malformed JSON request | `axis_chaos_test.py::malformed_json_controlled_rejection` | Bad request payloads must receive controlled client errors. | PASS | No WAF/rate-limit layer. |
| Huge payload | `axis_chaos_test.py::huge_payload_survives` | Oversized or pathological payloads must not produce unsafe allow and must not permanently break service. | PASS | Long-running resource exhaustion testing is not complete. |
| Concurrent approval race | `axis_chaos_test.py::concurrent_approval_resolve_race` | A single approval must not execute more than once; only one final decision should win. | PASS | Single-process local store only; no distributed lock model. |
| Restart during traffic | `axis_chaos_test.py::restart_during_traffic` | Risky SQL must not receive unsafe `200 ALLOW` during restart pressure. | PASS | No rolling-deploy or multi-instance behavior validated. |
| DB pool pressure | `axis_chaos_test.py::db_pool_pressure` | Pool pressure must not produce unsafe allow for blocked or approval-required requests. | PASS | No production SLO or saturation metrics. |
| Corrupt final audit entry at startup | `axis_chaos_test.py::corrupt_audit_startup_fail_fast` | AXIS must fail startup when the final audit entry cannot be parsed for chain recovery. | PASS | Full historical chain verification is not implemented. |
| Audit-chain restart continuity | `axis_audit_restart_test.py` | First post-restart event `previous_hash` must equal pre-restart last `event_hash`. | PASS | Verifies continuity at restart boundary only, not whole history. |
| Parser bypass corpus | `axis_regression.py` with `tests/parser_bypass_cases.json` | Risky parser bypass cases must not result in unsafe `200 ALLOW`. | PASS | Corpus is finite and should keep growing. |
| Policy regression | `axis_regression.py` with `tests/policy_cases.json` | Read/write/block/approval decisions must match policy expectations. | PASS | Policy coverage is for current `prod_main` baseline only. |
| Approval regression | `axis_regression.py` with `tests/approval_cases.json` | Approval-required rules must create approval flow outcomes. | PASS | No authenticated approver identity in v1. |
| Stress baseline | `axis_gate_stress.py` | 1000 requests at 50 concurrency must finish with unexpected result count 0. | PASS | Not a long-running soak test. |

## Failure Policy

The v1 safety policy is fail-safe for write paths:

- If SQL cannot be parsed or classified safely, reject it.
- If policy cannot be loaded, do not start normally.
- If audit evidence cannot be written, do not execute writes.
- If approval state is corrupt, do not approve or execute from that state.
- If PostgreSQL is unavailable, do not convert risky writes into `ALLOW`.

## Not Yet Covered

- Authenticated caller and approver identity failure modes.
- TLS termination and certificate failure modes.
- RBAC denial paths.
- Log rotation and retention failures.
- Full audit-chain replay verification failures.
- Multi-instance split-brain or distributed audit-chain conflicts.
- Long-duration soak, disk pressure, memory pressure, and process supervisor behavior.
# AXIS Install Guide

This guide is for reviewers running AXIS locally for technical evaluation. It avoids production deployment assumptions.

## Prerequisites

Windows:

- Git
- Rust toolchain from `rustup`
- Docker Desktop, recommended for the bundled PostgreSQL path
- Node.js LTS
- npm

Linux/macOS:

- Git
- Rust toolchain from `rustup`
- Docker and Docker Compose, recommended for the bundled PostgreSQL path
- Node.js LTS
- npm

## Clone

```cmd
git clone <repository-url>
cd AXIS
```

If you already have the repository locally, run commands from the repository root.

## Backend Setup

Run static and test checks first:

```cmd
cargo fmt --check
cargo check
cargo test
```

For local review with the included PostgreSQL container:

```cmd
docker compose up --build
```

This starts:

- AXIS on `http://localhost:6543`
- PostgreSQL on `localhost:5432`

To run the Rust service directly instead of Docker Compose, PostgreSQL must already be reachable:

```cmd
cargo run
```

## Environment Variables

Backend environment variables are loaded from the process environment and `.env` through `dotenvy`.

| Variable | Default | Purpose |
| --- | --- | --- |
| `LISTEN_ADDR` | `0.0.0.0:6543` | AXIS bind address and port |
| `DATABASE_URL` | Built from DB fields below | PostgreSQL connection string used by `tokio-postgres` |
| `DB_HOST` | `localhost` | PostgreSQL host when `DATABASE_URL` is not set |
| `DB_PORT` | `5432` | PostgreSQL port when `DATABASE_URL` is not set |
| `DB_NAME` | `prod_main` | PostgreSQL database name and default classifier database |
| `DB_USER` | `varux` | PostgreSQL user when `DATABASE_URL` is not set |
| `DB_PASS` | `varux` | PostgreSQL password when `DATABASE_URL` is not set |
| `AXIS_POLICY_DIR` | `./policies` | Directory containing deployable policy files |
| `AXIS_POLICY_MANIFEST` | `./policies/policy_manifest.json` | Authoritative startup manifest for the active policy |
| `AXIS_ENABLE_POLICY_RELOAD` | `false` | Enables internal controlled reload hooks; no HTTP reload endpoint is exposed |
| `POLICY_PATH` | `./policies/prod_main.json` | Deprecated compatibility variable; startup uses `AXIS_POLICY_MANIFEST` |
| `AXIS_POLICY_STORE_PATH` | `./data/policies` | Local policy lifecycle store |
| `AUDIT_WAL_PATH` | `./audit.wal` | Durable audit WAL path |
| `AUDIT_LOG_PATH` | `./audit.log` | JSONL audit projection used by visibility and verification endpoints |
| `AXIS_APPROVAL_DB_PATH` | `./data/approvals.sqlite` | SQLite approval state machine store |
| `OPERATING_MODE` | `enforce` | One of `shadow`, `approval_first`, `enforce`, `emergency_bypass` |
| `AXIS_OPERATOR_TOKEN` | unset | Required token for mutating policy lifecycle endpoints |

These endpoints require `AXIS_OPERATOR_TOKEN` to be configured and a matching `X-AXIS-Operator-Token` header:

- `POST /policy/candidates`
- `POST /policy/activate`
- `POST /policy/rollback`

Control plane variables:

| Variable | Default | Purpose |
| --- | --- | --- |
| `AXIS_CONTROL_PLANE_MODE` | `real` | Use `real` for backend integration or `mock` for explicit UI demo mode |
| `AXIS_BACKEND_URL` | `http://localhost:6543` | Server-side backend target used only by the Next.js proxy |
| `AXIS_PROXY_TIMEOUT_MS` | `8000` | Server-side proxy timeout |
| `NEXT_PUBLIC_REFRESH_INTERVAL_MS` | `5000` | Control plane polling interval |

## Frontend Setup

```cmd
cd control-plane
npm install
npm run build
npm run dev
```

Open:

```text
http://localhost:3000
```

The control plane proxies browser requests through `/api/axis/...` to the server-side `AXIS_BACKEND_URL`. The browser must not call the backend directly. The default backend is:

```text
http://localhost:6543
```

## Smoke Test

Use `curl.exe` on Windows CMD or PowerShell.

```cmd
curl.exe http://localhost:6543/health
curl.exe http://localhost:6543/runtime/stats
curl.exe "http://localhost:6543/logs?limit=10"
curl.exe http://localhost:6543/evidence/verify
```

Run a safe read:

```cmd
curl.exe -X POST http://localhost:6543/query -H "Content-Type: application/json" -d "{\"actor\":\"reviewer\",\"app\":\"demo\",\"tenant\":\"acme\",\"role\":\"developer\",\"host\":\"localhost\",\"env\":\"prod\",\"sql\":\"SELECT 1 AS axis_demo\"}"
```

Run a risky write that should be blocked by the default policy:

```cmd
curl.exe -X POST http://localhost:6543/query -H "Content-Type: application/json" -d "{\"actor\":\"reviewer\",\"app\":\"demo\",\"tenant\":\"acme\",\"role\":\"developer\",\"host\":\"localhost\",\"env\":\"prod\",\"sql\":\"DELETE FROM users\"}"
```

Inspect recent evidence:

```cmd
curl.exe "http://localhost:6543/audit?limit=10"
curl.exe http://localhost:6543/evidence/verify
```

## Troubleshooting

`npm` is not recognized:

Install Node.js LTS and reopen the terminal so `node` and `npm` are on `PATH`.

`cargo` or `rustup` is not recognized:

Install Rust from `rustup`, then reopen the terminal.

Port already in use:

AXIS defaults to `6543`, the control plane defaults to `3000`, and Docker PostgreSQL defaults to `5432`. Stop the conflicting service or change `LISTEN_ADDR` / frontend port as needed.

Docker is not running:

Start Docker Desktop or the Docker daemon before `docker compose up --build`.

Backend is not reachable from the control plane:

Check server-side `AXIS_BACKEND_URL`, then confirm `curl.exe http://localhost:6543/health` works. From the Control Plane proxy, confirm `curl.exe http://localhost:3000/api/axis/health` and `curl.exe "http://localhost:3000/api/axis/logs?limit=10"` work.

Policy file not found:

Check `POLICY_PATH` and confirm `policies/prod_main.json` exists. If using Docker Compose, the compose file mounts `./policies` into `/app/policies`.

Policy lifecycle store cannot be written:

Check `AXIS_POLICY_STORE_PATH`. The default `./data/policies` must be writable by the AXIS process.

Audit log malformed or evidence verification is invalid:

Run `GET /audit?limit=10` and inspect backend logs. Legacy or manually edited records can make verification report invalid; AXIS should not silently repair evidence.

Operator token missing:

If `AXIS_OPERATOR_TOKEN` is set, include `X-AXIS-Operator-Token` on mutating policy lifecycle requests. Read-only policy endpoints do not require the token.

Database connection fails on `cargo run`:

Use `docker compose up --build` for the bundled local database, or set `DATABASE_URL` / DB fields for your PostgreSQL instance.
# AXIS Operator Visibility Layer

AXIS v0.4 adds a read-only operator visibility layer over the existing enforcement core. The goal is to make runtime state, audit evidence, active policy, approvals, and evidence integrity inspectable without weakening the deterministic write-path controls.

## Why Visibility Matters

AXIS sits in front of production PostgreSQL and makes ALLOW, BLOCK, or APPROVAL_REQUIRED decisions before writes proceed. Operators need to answer four questions quickly:

- Is the service healthy?
- What policy is loaded?
- What evidence was recorded for recent decisions?
- Is the audit hash chain intact?

The v0.4 layer answers those questions with read-only endpoints and a control-plane UI backed by those endpoints.

## Backend Endpoints

### `GET /audit`

Returns recent audit events from the configured audit log path.

Query parameters:

- `limit`: optional event limit, default `100`, maximum `500`

Response fields:

- `ok`: request success flag
- `events`: recent audit events, newest first
- `count`: number of events returned
- `limit`: applied limit
- `malformed_count`: non-empty audit lines that were not JSON objects
- `order`: currently `newest_first`

If the audit log file does not exist, AXIS returns an empty event list.

The visibility response preserves fingerprints, normalized SQL, hashes, and event metadata. Raw SQL text in the embedded raw event view is redacted for operator safety.

### `GET /audit/:event_hash`

Fetches a single audit event by `event_hash`.

Responses:

- `200` with `{ "ok": true, "event": ... }` when found
- `404` when no audit event has that hash

Malformed audit lines are skipped during lookup.

### `GET /runtime/stats`

Returns dashboard-oriented runtime status:

- service name and status
- current timestamp
- uptime in seconds
- operating mode
- audit path, existence, event count, malformed count, and last event hash
- policy load status, version, path, and rule count
- pending approval count
- decision counts derived from audit events

Decision counts are derived from recorded audit decisions, not synthetic counters.

### `GET /policy`

Returns the active policy as loaded by the backend, plus:

- policy version
- policy path
- rule count
- operating mode

This endpoint is read-only. Policy editing is intentionally not enabled in v0.4.

### `GET /policy/status`

Returns a lightweight policy health summary:

- `loaded`
- `version`
- `path`
- `rules_count`

### `GET /evidence/verify`

Verifies audit evidence integrity without mutating or repairing the log.

Response fields:

- `valid`: whether verification passed
- `verification_depth`: `full_hash` when AXIS recomputed event hashes, `linkage_only` if a future fallback only checks links
- `checked_events`: events successfully verified before the first invalid record
- `malformed_count`: malformed records encountered
- `first_invalid_index`: zero-based event index for the first invalid record
- `first_invalid_line`: physical line number for the first invalid record
- `first_invalid_reason`: human-readable failure reason
- `last_event_hash`: last successfully verified event hash

The current implementation uses `full_hash` verification for AXIS audit events. It checks:

- each event has `event_hash`
- each event has `previous_hash`
- `previous_hash` links to the previous event hash
- recomputed canonical event hash matches `event_hash`

If the configured log contains legacy records without hash-chain fields, verification reports them honestly as invalid/malformed instead of hiding the issue.

## Control Plane Usage

The Next.js control plane consumes the v0.4 endpoints directly:

- Dashboard: `/health`, `/runtime/stats`, `/evidence/verify`
- Evidence Explorer: `/audit?limit=100`
- Policy Viewer: `/policy`, `/policy/status`
- Runtime: `/runtime/stats`, `/evidence/verify`
- Approval Center: existing `/approvals` and `/approvals/:approval_id/resolve`
- Query Console: existing `/query`

The UI remains an operator surface. It does not create fake production metrics when real endpoints are available.

## Intentionally Not Included

v0.4 does not add:

- policy editing
- multi-user auth or RBAC
- SaaS deployment management
- remote agent systems
- distributed audit-chain coordination
- log rotation or archival

Those are candidates for future versions, with policy editing and auth belonging behind stronger operator identity controls.
# AXIS Policy Lifecycle & Trust Layer

AXIS v0.5 adds controlled policy lifecycle management around the existing deterministic policy engine. The write-path enforcement model is unchanged: production SQL still flows through the classifier, policy evaluator, approval store, executor, and audit evidence path already used by AXIS.

The lifecycle layer exists to make policy change a security-sensitive operation instead of a direct edit to production behavior.

## What v0.5 Adds

- Candidate policy validation before activation.
- Active-versus-candidate policy diffing.
- Dry-run decision previews for current or candidate policy.
- Immutable policy version files with a manifest.
- Safe candidate activation with expected-hash checks.
- Rollback to a previous valid stored policy.
- Control-plane views for validation, diff, dry-run, versions, activation, and rollback.
- Optional operator token protection for mutating lifecycle endpoints.

## Why Policy Lifecycle Matters

A policy file controls the production write path. A bad policy can accidentally allow broad writes, remove approval gates, or weaken DDL restrictions. AXIS v0.5 makes each policy change answer these questions:

- Is the policy structurally valid?
- Does it contain risky semantic patterns?
- What changed from the active policy?
- What would representative SQL do under the candidate?
- Is the exact stored candidate hash being activated?
- Can the operator roll back to a known-good version?

## Validation Model

`POST /policy/validate` validates a supplied JSON policy without mutating runtime state.

Structural checks include:

- Policy is a JSON object.
- `policy_version` or `version` exists.
- `write_rules` exists and is an array.
- Rule IDs are present and unique.
- Action values are valid: `ALLOW`, `BLOCK`, `REQUIRE_APPROVAL`, or `APPROVAL_REQUIRED`.
- Required typed fields deserialize through the existing policy model.

Semantic checks currently warn on:

- Dangerous production wildcard `ALLOW` rules.
- Default write or DDL behavior that is too permissive.
- Unknown SQL shape fallback behavior that is not a hard block.
- Missing DELETE-without-WHERE, batch UPDATE, batch DELETE, or DDL guards.
- Duplicate match conditions.
- Shadowed rules where an earlier rule can match before a later one.
- Broad migration-oriented production allow rules.
- Unknown environment labels.

Validation is read-only. It does not reload policy, execute SQL, create approvals, or write production audit evidence.

## Diff Model

`POST /policy/diff` compares the active policy with a candidate policy.

The diff reports:

- Added rules.
- Removed rules.
- Modified rules.
- Action changes.
- Match scope changes.
- Approval and reason-code changes.
- Risk level for each change and for the whole diff.

Rules are matched by rule ID. If IDs are absent, AXIS falls back to a normalized JSON fingerprint. Stable rule IDs are recommended for operator-readable diffs.

Risk is conservative. Removing production protections or changing `BLOCK` / `REQUIRE_APPROVAL` to `ALLOW` is treated as higher risk.

## Dry-Run Model

`POST /policy/dry-run` previews a decision under either the current active policy or a supplied candidate policy.

Dry-run reuses the existing SQL classifier and policy evaluator. It returns:

- Decision.
- Reason.
- SQL fingerprint.
- Classification details.
- Matched rule trace.
- `would_execute: false`.
- `audit_written: false`.

Dry-run never executes SQL, never writes audit evidence, and never creates approval requests. Unsupported or unparsable SQL fails safe with a blocking preview.

## Version Store Model

The default local store is:

```text
data/policies/
  active.json
  manifest.json
  versions/
    policy-vYYYYMMDD-HHMMSS-xxxxxxxx.json
```

Each immutable version document contains:

- Version record metadata.
- The policy JSON.
- The validation result at creation time.

The manifest tracks:

- Version ID.
- Policy version string.
- Hash.
- Created and activated timestamps.
- Status: `active`, `candidate`, `archived`, or `rejected`.
- Rule count.
- Validation status.
- Source.
- Optional operator note.

Version files are created with create-new semantics and are not rewritten. The manifest and active pointer are updated through temp-file writes and replacement with backup recovery where the platform allows it.

## Activation Safety Model

`POST /policy/activate` activates a stored candidate only when:

- The candidate exists.
- Its status is `candidate`.
- `expected_hash` matches the stored hash.
- The stored policy file hash still matches the manifest hash.
- The policy validates again immediately before activation.

Activation writes `active.json`, updates the manifest status, and swaps the in-memory active policy handle. If validation or hash verification fails, AXIS keeps the previous active policy.

AXIS does not delete old versions during activation.

## Rollback Model

`POST /policy/rollback` activates a previous valid stored version by ID or policy version string.

Rollback requires:

- Target version exists.
- Target is not rejected.
- `expected_hash` matches.
- Target policy validates before activation.

The current active version becomes archived and the target becomes active.

## Operator Token Behavior

AXIS v0.5 supports a minimal optional operator token:

```text
AXIS_OPERATOR_TOKEN=...
```

When configured, mutating lifecycle endpoints require:

```text
X-AXIS-Operator-Token: ...
```

Protected endpoints:

- `POST /policy/candidates`
- `POST /policy/activate`
- `POST /policy/rollback`

Read-only lifecycle endpoints remain open for local visibility. If no token is configured, local development remains unblocked and `/policy/status` reports `operator_auth_enabled: false`.

## Intentionally Not Included

AXIS v0.5 does not include:

- Full RBAC.
- SaaS control plane.
- Remote policy distribution.
- Multi-instance consensus.
- Automatic policy optimization.
- AI-generated policy activation.

Those are future trust and deployment concerns. v0.5 focuses on safe local lifecycle control for one AXIS runtime.
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

- No auth, TLS, or RBAC.
- No metrics or observability.
- No log rotation.
- Recovery procedures are manual and initial-level.
- No full historical audit-chain verifier.
- No multi-instance distributed audit chain.
- No long-running soak test.
- No production deployment hardening.
# AXIS Security Model

AXIS v1 is not a production-ready enterprise product. It is a local hardened v1 security core. The model below describes the current local security boundary and evidence assumptions only.

## Purpose

AXIS protects database execution by placing a deterministic gate before SQL reaches PostgreSQL. The gate classifies SQL, evaluates a versioned policy, writes audit evidence, and then either executes the SQL, blocks it, or creates an approval request.

## Assets Protected

- PostgreSQL data reachable through the AXIS `/query` endpoint.
- Policy-controlled write and DDL paths.
- Approval records in `approvals.jsonl`.
- Audit evidence in `audit.log`.
- Policy file integrity at startup.

## Trust Boundary

Trusted local components:

- AXIS process.
- Local policy file configured by `POLICY_PATH`.
- Local audit log path configured by `AUDIT_LOG_PATH`.
- Local approval store path configured by `APPROVAL_STORE_PATH`.
- PostgreSQL instance configured by `DATABASE_URL` or DB environment variables.

Untrusted or semi-trusted inputs:

- HTTP request body sent to `/query`.
- SQL text in the request.
- Request identity fields such as `actor`, `app`, `tenant`, `role`, `host`, and `env`.
- Approval resolution requests sent to `/approvals/{approval_id}/resolve`.

Important limitation: v1 does not authenticate these HTTP callers. Identity fields are evidence and policy inputs, not verified identity claims.

## Request Flow

1. Request reaches `POST /query`.
2. AXIS validates required request fields and size limits.
3. AXIS parses SQL using a PostgreSQL dialect parser.
4. Multiple statements and unsupported write-like read forms are rejected.
5. AXIS derives operation, target, scope estimate, normalized SQL, SQL fingerprint, and risk signals.
6. Policy is evaluated in the current operating mode.
7. Audit evidence is written before execution-sensitive outcomes.
8. Enforcement returns one of:
   - `200 ALLOW`
   - `403 BLOCK`
   - `202 REQUIRE_APPROVAL`
   - controlled `4xx` or `5xx` failure

## Policy Model

The current policy is `policies/prod_main.json`.

Policy behavior:

- Reads default to `ALLOW`.
- Writes default to `BLOCK`.
- DDL defaults to `REQUIRE_APPROVAL`.
- Specific single-row `orders` updates by allowed `webapp` roles can be allowed.
- Bulk updates require approval.
- Deletes without `WHERE` are blocked.
- Selected migration cleanup paths require approval with a different approver group.

Policy decisions are represented as:

- `ALLOW`
- `BLOCK`
- `REQUIRE_APPROVAL`

Operating modes:

- `shadow`: records policy result but allows execution.
- `approval_first`: can force high-risk writes to approval.
- `enforce`: applies policy decisions.
- `emergency_bypass`: can allow blocked write paths with audit evidence.

The current hardening evidence uses `enforce`.

## Parser and Classifier Controls

Current controls include:

- Empty SQL rejection.
- Multiple-statement rejection.
- Parser errors converted to controlled request rejection.
- Unsupported statement types rejected.
- Write-like or locking read forms rejected, including selected CTE and `SELECT INTO` patterns.
- SQL normalization and fingerprinting for audit evidence.
- Scope estimation for single-row versus batch write behavior.
- Risk signals such as `prod_write`, `ddl_operation`, `bulk_operation`, `delete_without_where`, `unknown_target`, `unknown_scope`, and `cross_schema`.

The parser bypass corpus confirms that risky SQL does not get unsafe `200 ALLOW` under the tested baseline.

## Approval Model

Approval records are stored in append-only JSONL form at `APPROVAL_STORE_PATH`.

Approval flow:

- A policy decision of `REQUIRE_APPROVAL` creates a pending approval record.
- The original query is not executed when the approval is created.
- A resolution request can approve or reject the pending approval.
- Approved requests are executed after audit writability is checked.
- Rejected requests are blocked.
- Already resolved approvals cannot be resolved again.

The concurrent approval race test verified that one approval cannot safely produce multiple final executions in the tested local baseline.

Limitation: v1 does not authenticate approvers or enforce RBAC. The `actor` in approval resolution is recorded, not cryptographically verified.

## Audit Model

Audit events are written as JSON lines to `AUDIT_LOG_PATH`.

Each event includes:

- Event id and request id.
- Event type.
- Request identity fields.
- SQL evidence: operation, classifier, fingerprint, normalized SQL, raw SQL, params.
- Target and scope.
- Risk signals.
- Decision and policy decision.
- Reason code and policy version.
- `previous_hash`.
- `event_hash`.

The hash chain is local and append-oriented. On startup, AXIS reads the final non-empty audit log entry and uses its `event_hash` as the next event's `previous_hash`.

Current evidence proves restart continuity from the last event into the first post-restart event. It does not prove full historical chain verification from the first event to the last event.

## Fail-Safe Behavior

The current local baseline verifies:

- PostgreSQL down does not produce unsafe allow for risky writes.
- Audit unwritable state does not execute writes without evidence.
- Invalid policy JSON prevents healthy startup.
- Corrupt approval store fails safely.
- Corrupt final audit entry prevents healthy startup.
- Malformed JSON and oversized inputs receive controlled rejection.
- Restart during traffic does not allow risky SQL unsafely under the tested load.

## Explicit Non-Goals in v1

- No auth, TLS, or RBAC.
- No production-grade tenant isolation.
- No external secrets management.
- No metrics or observability backend.
- No log rotation or retention policy.
- No full historical audit-chain verifier.
- No distributed audit-chain coordination across multiple AXIS instances.
- No production deployment hardening.
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
# AXIS Threat Model

## Assets Protected

- Production database write path.
- Policy integrity.
- Audit evidence integrity.
- Approval workflow integrity.
- Operator control plane.
- Policy lifecycle store.

## Threat Actors

- Compromised application credential.
- Buggy internal tool.
- Malicious insider.
- Careless operator.
- Compromised migration script.
- Attacker with partial network access.
- Attacker trying to tamper with logs.
- Attacker trying to weaken policy.

## Threat Scenarios

| Scenario | Risk | Current AXIS response |
| --- | --- | --- |
| A. Destructive `DELETE` reaches production | Data loss | SQL classification detects `DELETE`, risk signals include `delete_without_where` when applicable, policy can `BLOCK` or require approval |
| B. `UPDATE` without `WHERE` | Broad unintended mutation | Scope estimation marks batch writes; default policy requires approval for batch updates |
| C. DDL in production | Schema damage or outage | DDL classification and default DDL policy require approval |
| D. Hidden mutation through unsupported SQL shape | Parser bypass | Unsupported or write-like SQL shapes fail safe instead of being treated as safe reads |
| E. Approval bypass attempt | Risky write executes before review | `REQUIRE_APPROVAL` creates a pending approval and does not execute until resolution |
| F. Audit log tampering | Evidence cannot be trusted | Hash-chain verification reports malformed records, missing hashes, and broken links |
| G. Policy downgrade | Broad writes become allowed | Validation and diff highlight risky changes before activation |
| H. Candidate policy activated without validation | Invalid policy controls write path | Candidate creation and activation validate policy; invalid candidates are rejected |
| I. Rollback to malicious policy | Old bad policy becomes active | Rollback validates target policy and checks expected hash |
| J. Operator endpoint misuse | Unauthorized lifecycle mutation | Optional `AXIS_OPERATOR_TOKEN` protects candidate creation, activation, and rollback |
| K. Frontend/backend mismatch | Operator sees stale or wrong state | Control plane reads live endpoints in real mode and surfaces backend errors |
| L. Local policy store corruption | Runtime loads inconsistent policy | Store reads validate records and policy activation keeps previous active policy on failure paths |

## Mitigations Currently Implemented

- Policy enforcement for `ALLOW`, `BLOCK`, and `REQUIRE_APPROVAL`.
- Unsupported SQL fail-safe behavior through parser/classifier rejection.
- Approval workflow that records pending requests and prevents execution until approval.
- Audit hash chain with `previous_hash` and `event_hash`.
- Restart continuity from the last recorded audit event hash.
- Evidence verification through `GET /evidence/verify`.
- Policy validation for structural and semantic policy issues.
- Policy diff to highlight added, removed, modified, and risky rules.
- Policy dry-run that previews decisions without execution or mutation.
- Expected-hash activation for stored candidates.
- Rollback validation and expected-hash checks.
- Optional operator token for mutating lifecycle endpoints.

## Known Gaps

- No full RBAC.
- No SSO.
- No TLS/mTLS story in the v0.6 package.
- No distributed consensus.
- No external KMS.
- No tamper-proof external ledger.
- No formal compliance mapping.
- Local JSON/file-backed policy store.
- No dedicated operator audit stream yet.
- Request identity fields are not cryptographically verified by AXIS itself.
- No multi-instance policy synchronization.

## Pilot Risk Posture

AXIS v0.6 is suitable for:

- Local evaluation.
- Controlled demo.
- Technical review.
- Non-production pilot planning.
- Staging or lab validation against representative SQL flows.

AXIS v0.6 is not yet suitable for:

- Unsupervised production deployment.
- Regulated deployment without additional controls.
- Multi-region enterprise rollout.
- Deployment where AXIS is exposed without network, identity, and transport protections.

## Review Focus

Reviewers should pay particular attention to:

- SQL parser and classifier coverage for their query patterns.
- Whether policy defaults are conservative enough for the target environment.
- Whether approval records and audit evidence meet local evidence requirements.
- Whether local file-backed state is acceptable for a pilot.
- Which identity, TLS, KMS, retention, and operator audit controls must exist before production use.
# AXIS v1 Hardening Report

Status date: 2026-05-10

AXIS v1 is not a production-ready enterprise product. It is a local hardened v1 security core for deterministic SQL gate enforcement, policy regression, approval flow validation, audit evidence, and failure-mode testing.

## Current Verdict

Hardening phase: ACTIVE+

| Area | Verdict |
| --- | --- |
| AXIS v1 runtime | PASS |
| Parser bypass corpus baseline | PASS |
| Policy regression baseline | PASS |
| Approval regression baseline | PASS |
| Audit-chain restart continuity | PASS |
| Corrupt audit fail-fast | PASS |
| Chaos/failure-mode baseline | PASS |

## What Was Proven

The current v1 baseline proves the following behavior in the local hardened core:

- Read, write, block, and approval paths are working.
- Risky SQL does not receive unsafe `200 ALLOW`.
- Parser bypass corpus baseline passed.
- Policy regression baseline passed: 31/31 total regression cases.
- Approval regression baseline passed.
- Stress baseline passed with 1000 requests, 50 concurrency, and unexpected result count 0.
- Audit chain continuity across restart is preserved: the first post-restart `previous_hash` continues from the previous event `event_hash`.
- Corrupt audit startup fail-fast behavior is verified.
- Chaos suite passed: 10/10 scenarios.
- The tested chaos scenarios include PostgreSQL down, audit unwritable, invalid policy, corrupt approval store, malformed JSON, huge payload, concurrent approval race, restart during traffic, DB pool pressure, and corrupt audit startup.

## Security Boundary

AXIS v1 sits in front of the database write path and evaluates incoming SQL before execution. It classifies SQL, applies the configured policy, writes audit evidence, and returns one of these outcomes:

- `ALLOW`: query is executed and audited.
- `BLOCK`: query is not executed and the block is audited.
- `REQUIRE_APPROVAL`: query is queued for approval and not executed until a valid approval resolution occurs.

The current policy file is `policies/prod_main.json`. The default operating mode in local configuration is `enforce`.

The v1 guarantee is local and process-bound. It does not claim distributed, multi-instance, production-grade audit finality or enterprise access control.

## Evidence Sources

Primary local evidence sources:

- `axis_regression.py`
- `axis_gate_stress.py`
- `axis_audit_restart_test.py`
- `axis_chaos_test.py`
- `tests/policy_cases.json`
- `tests/parser_bypass_cases.json`
- `tests/approval_cases.json`
- `policies/prod_main.json`
- `audit.log`
- `approvals.jsonl`

Main validation commands are documented in `docs/technical/AXIS_TEST_COMMANDS.md`.

## Interpreted Result

The local v1 hardening work is sufficient to call AXIS a hardened security core for local pre-execution SQL gate validation. The evidence supports deterministic policy behavior, controlled rejection for malformed input, approval-store behavior under race pressure, audit-chain restart continuity, and fail-safe behavior under the tested chaos cases.

This does not make AXIS production-ready. It means the v1 core has a meaningful evidence baseline and known failure behavior.

## Remaining Risks

The following risks remain open and should be treated as explicit non-goals or future hardening work:

- No auth, TLS, or RBAC.
- No metrics or observability pipeline.
- No log rotation.
- Recovery runbook is initial-level only.
- No full historical audit-chain verification.
- No multi-instance distributed audit chain.
- No long-running soak test.
- No production deployment hardening.

## README Current Status Draft

The following short section can be added to `README.md`:

```markdown
## Current Status

AXIS v1 is a local hardened v1 security core, not a production-ready enterprise product.

Current baseline: runtime PASS, parser bypass corpus PASS, policy regression 31/31 PASS, approval regression PASS, audit-chain restart continuity PASS, corrupt audit startup fail-fast PASS, chaos/failure-mode suite 10/10 PASS.

Known gaps remain: auth/TLS/RBAC, metrics/observability, log rotation, full historical audit-chain verification, distributed audit-chain support, long-running soak testing, and production deployment hardening.
```
# DB Access Hardening

AXIS runtime must not use PostgreSQL superuser credentials. Enterprise deployments should separate migration, AXIS execution, and application read-only access.

## Roles

`migration_admin`

Owns the enterprise demo schema and is used only for schema setup and migrations. This role is not used by AXIS runtime.

`axis_executor`

Runtime role used by AXIS. It is `NOSUPERUSER`, does not own `prod_main`, has schema `USAGE`, and has scoped DML privileges on the enterprise demo tables. It does not receive broad DDL privileges.

`app_readonly`

Read-only proof role. It can `SELECT` permitted demo tables and cannot `INSERT`, `UPDATE`, `DELETE`, `DROP`, `ALTER`, or `TRUNCATE`. This role is used by `scripts/enterprise_boundary_check.py` to prove direct app write bypass fails.

## Apply Roles

The enterprise compose profile applies:

```text
sql/enterprise_roles.sql
sql/enterprise_seed.sql
```

on first PostgreSQL volume initialization.

For an existing database, review the scripts first, then apply through a controlled migration path as a bootstrap PostgreSQL administrator. Do not run AXIS as that administrator.

## Verify Grants

Run:

```bash
python scripts/enterprise_boundary_check.py
```

The check verifies:

- PostgreSQL is not exposed on host port 5432
- `app_readonly` cannot write directly
- `axis_executor` is not superuser
- `axis_executor` does not own `prod_main`

Manual checks:

```sql
SELECT rolname, rolsuper, rolcreatedb, rolcreaterole
FROM pg_roles
WHERE rolname IN ('axis_executor', 'app_readonly', 'migration_admin');

SELECT grantee, privilege_type
FROM information_schema.role_table_grants
WHERE grantee IN ('axis_executor', 'app_readonly')
ORDER BY grantee, table_name, privilege_type;
```

## Credential Rotation

The compose file uses demo passwords only. Production deployments should source credentials from a secret manager, rotate `axis_executor` and application credentials independently, and revoke any direct write-capable application credentials before routing protected writes through AXIS.
# AXIS Deployment

This document covers local reviewer deployment for AXIS with Docker Compose. It is not a managed production runbook.

## Prerequisites

- Docker Desktop or Docker Engine with Compose v2.
- Rust toolchain for local backend validation.
- Python 3.10+ for smoke, verifier, regression, restart, chaos, and stress scripts.
- Node.js 20+ for the Control Plane.

Run commands from the repository root unless noted.

## Local Demo Startup

The default Compose mode starts PostgreSQL and the AXIS backend on `http://localhost:6543`. Port `6544` also maps to the same backend for validation scripts that need to avoid a stale local listener.

```powershell
docker compose down
docker compose up --build -d
docker compose ps
```

Check the runtime:

```powershell
curl.exe http://localhost:6543/health
curl.exe "http://localhost:6543/runtime/metrics"
curl.exe "http://localhost:6543/audit/verify"
```

The default profile is `local`. It uses a local-only placeholder operator token and unsigned Evidence Bundle V1 export.

## Demo Profile

The demo profile is the default local Compose deployment with:

- PostgreSQL service `postgres`.
- AXIS backend service `dbguard`.
- Policy mounted read-only from `./policies`.
- Audit WAL, JSONL projection, approvals, and policy store held in named volume `dbguard_data`.
- Local export auth disabled by default with `AXIS_AUDIT_EXPORT_REQUIRES_OPERATOR=false`.

Run smoke and stress checks:

```powershell
python scripts/axis_audit_api_smoke.py --base http://localhost:6543
python scripts/axis_runtime_smoke.py --base http://localhost:6543
python scripts/axis_runtime_stress.py --base http://localhost:6543 --concurrency 25 --requests 500 --include-export --include-rate-limit
```

## Production-Like Profile

Production-like mode uses the `dbguard-production-like` Compose service and publishes `http://localhost:6545`. It sets `AXIS_RUNTIME_PROFILE=production`, requires a strong `AXIS_OPERATOR_TOKEN`, and defaults audit export auth to enabled.

Create a private env file from `.env.production.example` and replace placeholders:

```powershell
Copy-Item .env.production.example .env.production.local
notepad .env.production.local
docker compose --env-file .env.production.local --profile production-like up --build -d postgres dbguard-production-like
docker compose --profile production-like ps
```

Validate:

```powershell
curl.exe http://localhost:6545/health
curl.exe "http://localhost:6545/runtime/metrics"
curl.exe -H "X-AXIS-Operator-Token: <your-token>" "http://localhost:6545/audit/export?limit=5" -o axis-evidence-bundle-v1.json
python scripts/axis_evidence_bundle_verify.py --bundle axis-evidence-bundle-v1.json
```

Do not commit `.env.production.local`.

## Environment Variables

Required or commonly reviewed backend variables:

- `AXIS_RUNTIME_PROFILE`: `local` or `production`.
- `AXIS_OPERATOR_TOKEN`: required and strong when profile is `production`.
- `AXIS_REQUEST_TIMEOUT_MS`, `AXIS_DB_QUERY_TIMEOUT_MS`, `AXIS_DB_CONNECT_TIMEOUT_MS`.
- `AXIS_DB_POOL_MAX_CONNECTIONS`, `AXIS_DB_POOL_ACQUIRE_TIMEOUT_MS`.
- `AXIS_RATE_LIMIT_ENABLED`, `AXIS_RATE_LIMIT_REQUESTS_PER_MINUTE`, `AXIS_RATE_LIMIT_BURST`.
- `AXIS_MAX_BODY_BYTES`, `AXIS_MAX_SQL_BYTES`.
- `AXIS_EVIDENCE_SIGNING_ENABLED`, `AXIS_EVIDENCE_SIGNING_KEY_ID`, `AXIS_EVIDENCE_SIGNING_PRIVATE_KEY_B64`, `AXIS_EVIDENCE_SIGNING_PUBLIC_KEY_B64`.
- `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASS`, `DB_NAME`, or `DATABASE_URL`.
- `AXIS_POLICY_DIR`, `AXIS_POLICY_MANIFEST`, `POLICY_PATH`.
- `AUDIT_WAL_PATH`, `AUDIT_LOG_PATH`, `AUDIT_INDEX_PATH`, `AXIS_APPROVAL_DB_PATH`, `AXIS_POLICY_STORE_PATH`.

`.env.example` is local-only. `.env.production.example` is a placeholder template and contains no real secrets.

## Runtime Validation Commands

```powershell
cargo fmt
cargo check
cargo test

docker compose down
docker compose up --build -d
docker compose ps

curl.exe http://localhost:6543/health
curl.exe "http://localhost:6543/runtime/metrics"
curl.exe "http://localhost:6543/audit/events?limit=5"
curl.exe "http://localhost:6543/audit/verify"
curl.exe "http://localhost:6543/audit/export?limit=5" -o axis-evidence-bundle-v1.json

python scripts/axis_evidence_bundle_verify.py --bundle axis-evidence-bundle-v1.json
python scripts/axis_audit_api_smoke.py --base http://localhost:6543
python scripts/axis_runtime_smoke.py --base http://localhost:6543
python scripts/axis_runtime_stress.py --base http://localhost:6543 --concurrency 25 --requests 500 --include-export --include-rate-limit
python axis_regression.py --base http://localhost:6543
python axis_audit_restart_test.py --base http://localhost:6543
python axis_chaos_test.py --base http://localhost:6543
```

Control Plane:

```powershell
cd control-plane
npm run typecheck
npm run lint
npm run build
npm run smoke:real
npx playwright test e2e/real-mode.spec.ts
```

## Evidence Export

Export a small bundle:

```powershell
curl.exe "http://localhost:6543/audit/export?limit=10" -o axis-evidence-bundle-v1.json
python scripts/axis_evidence_bundle_verify.py --bundle axis-evidence-bundle-v1.json
```

If signing is enabled, pass the public key and require a signature:

```powershell
python scripts/axis_evidence_bundle_verify.py --bundle axis-evidence-bundle-v1.json --public-key-b64 <public-key> --require-signature
```

## Data Volume Behavior

`dbguard_data` stores `/app/data`, including:

- `audit.wal`: canonical WAL-backed audit evidence.
- `audit.log`: JSONL projection for operator convenience.
- `approvals.sqlite`: SQLite approval state machine store.
- policy store files.

`pgdata` stores PostgreSQL data. `docker compose down` stops services but keeps volumes. Reset local state with:

```powershell
docker compose down -v
```

This deletes local Docker volumes. Do not run it against an environment whose evidence you need to preserve.

## Policy Manifest Behavior

The active policy is loaded from `AXIS_POLICY_MANIFEST`, which points at `policies/policy_manifest.json` in Compose. The manifest checksum and policy version must match the active policy file. Startup fails fast on missing policy files, malformed policy JSON, checksum mismatch, or policy version mismatch.

## Operator Token Behavior

Policy mutation and approval resolution require operator auth. Audit export requires operator auth when `AXIS_AUDIT_EXPORT_REQUIRES_OPERATOR=true`, which is the production-profile default. Tokens are supplied server-side and are not exposed to browser code.

## Signing Behavior

Evidence signing is disabled by default. When enabled, AXIS expects base64 Ed25519 key material through server-only environment variables. Private keys must not be committed, copied into images, or exposed through frontend env variables.

## Known Limitations

- This Compose setup is local reviewer packaging, not a high-availability deployment.
- WAL-backed audit reads and exports use Audit Derived Index V1 for candidate selection when the index is ready. Event bodies, verification, and Evidence Bundle V1 integrity remain WAL-backed.
- Evidence Bundle V1 proves included AXIS evidence integrity, not external business truth.
- Production-like mode demonstrates fail-fast config and operator auth behavior but does not replace secret management, TLS termination, backups, or monitoring.
# AXIS Driver Matrix

Status date: 2026-07-05

This matrix tracks real-client compatibility for AXIS pgwire mode. Batch and
bulk behavior is listed separately because `PIPELINING_SUPPORT=false` rejects
extended-query cycles with more than one Bind/Execute pair before Sync.

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

pgjdbc batch execution (`addBatch`/`executeBatch`) is NOT SUPPORTED in MVP.
Requires `PIPELINING_SUPPORT=true` (future phase). Verified: batch is cleanly
rejected (`42501`/`unsupported_extended_pipeline`), connection
remains usable after rejection. Regression test
`test_jdbc_batch_execute_is_cleanly_rejected_not_hung_or_corrupted` covers this.

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
| PostgreSQL JDBC | Java | pgjdbc 42.7.7 batch traffic used Extended Query; default connection may send whitelisted Simple Query `SET application_name` during startup. | Covered by `tests/test_pgwire_real_drivers.py`; default connection without `assumeMinServerVersion=12` is covered. | Covered (single-row INSERT). | Covered (single-row UPDATE). | Covered, expects SQLSTATE `42501`. | Covered. | Covered; savepoint recovery after policy deny passed. | Covered with repeated prepared execution. | Batch execution (`addBatch`/`executeBatch`): NOT SUPPORTED in MVP. Requires `PIPELINING_SUPPORT=true` (future phase). Verified: batch is cleanly rejected (`42501`/`unsupported_extended_pipeline`), connection remains usable after rejection. Regression test added. | Same JDBC connection executed `SELECT 1` after each batch rejection. | Production leakage test not added yet. | Single-statement production tests implemented; batch explicitly excluded per policy decision. | `assumeMinServerVersion=12` is no longer required for startup. Batch/bulk APIs excluded from MVP compatibility surface. |

## CI Plan

Required jobs once the product decision gate is resolved:

| Job | Command | Notes |
| --- | --- | --- |
| Rust unit/integration | `cargo test --no-fail-fast` | Must always run. |
| Python real drivers | `python -m pytest tests/test_pgwire_real_drivers.py -vv -s` | Requires Docker daemon and PostgreSQL image access. |
| Prisma single-statement tests | `python -m pytest tests/test_pgwire_real_drivers.py -k prisma -vv -s` | Requires Docker, Node.js, and PostgreSQL image access. Batch/bulk excluded per policy decision. |
| JDBC single-statement tests | `python -m pytest tests/test_pgwire_real_drivers.py -k jdbc -vv -s` | Requires Docker, Java 21, and PostgreSQL image access. Batch explicitly rejected and regression-tested. |

Do not mark JDBC batch support green while `PIPELINING_SUPPORT=false`.

JDBC and Prisma single-statement test suites run in the same `real_pgwire`
fixture as psycopg3 and asyncpg. The JDBC tests invoke pgjdbc through a
subprocess Maven project at `tests/jdbc-axis-test/`. The Prisma tests use
`@prisma/client` generated against the proxy DSN.
# AXIS Enterprise Deployment Boundary

AXIS protects database operations that pass through AXIS. It cannot protect SQL sent directly to PostgreSQL with credentials or network access that bypass AXIS.

## Enforced Topology

```text
Application / Agent / Internal Tool
        |
        v
      AXIS
        |
        v
PostgreSQL private network
```

In the enterprise compose profile, AXIS is the only service with access to the private PostgreSQL network. PostgreSQL is not published on host port 5432.

```bash
docker compose -f docker-compose.enterprise.yml up --build
```

AXIS is exposed for local validation on:

```text
http://127.0.0.1:65431
```

Enterprise evidence is written to:

```text
./enterprise-data
```

Expected evidence includes `audit.log`, `audit.wal`, `approvals.sqlite` when approvals are used, and enterprise boundary reports.

## Blocked Topology

```text
Application / Agent / Internal Tool
        |
        v
PostgreSQL direct write access
```

This topology is a deployment failure for protected write paths. Production deployments must enforce private DB networking, firewall rules, security groups, IAM controls, and PostgreSQL role separation so application clients do not receive direct write-capable database credentials.

## Authenticated Request Context

When `AXIS_AUTH_MODE=jwt_hs256`, `/query` requires `Authorization: Bearer <token>`. AXIS derives actor, actor type, app, tenant, role, host, and environment from the verified token before policy evaluation and audit. Spoofed JSON body fields are ignored; conflicts are recorded in request audit payloads.

HS256 mode is for local enterprise validation only. Production identity should use OIDC/JWKS, mTLS, service identity, cloud IAM, or equivalent trusted identity infrastructure.
# Enterprise Runbook

## Start Enterprise Profile

```bash
docker compose -f docker-compose.enterprise.yml up --build
```

AXIS listens on:

```text
http://127.0.0.1:65431
```

PostgreSQL is internal-only and must not be published on host port 5432.

The Compose profile sets `AXIS_RUNTIME_PROFILE=local` and uses explicitly labeled local/demo secrets so boundary validation can run on a reviewer workstation. Do not reuse those values with `AXIS_RUNTIME_PROFILE=production`; production profile rejects local/demo/default secrets.

## Run Boundary Check

```bash
python scripts/enterprise_boundary_check.py
```

Expected success:

```text
AXIS ENTERPRISE BOUNDARY CHECK: PASS
```

Evidence path:

```text
./enterprise-data
```

Generated reports:

```text
./enterprise-data/enterprise-boundary-report.json
./enterprise-data/enterprise-boundary-report.md
```

## Troubleshooting

AXIS not healthy

Check `docker compose -f docker-compose.enterprise.yml ps`, AXIS logs, policy manifest validation, and PostgreSQL health.

PostgreSQL exposed on 5432

Remove `ports: 5432:5432` from the enterprise PostgreSQL service and keep it only on `axis_private_db`.

`app_readonly` can write

Reapply `sql/enterprise_roles.sql`, verify table ownership, and ensure no broad grants or role memberships give write privileges.

Missing or invalid JWT accepted

Verify `AXIS_AUTH_MODE=jwt_hs256`, `AXIS_JWT_REQUIRED=true`, and `AXIS_JWT_HS256_SECRET` match the boundary check environment.

Production JWT/auth validation

Set `AXIS_RUNTIME_PROFILE=production` only with strong non-default `AXIS_OPERATOR_TOKEN` and, if HS256 is temporarily enabled, a strong non-default `AXIS_JWT_HS256_SECRET`. For production IAM, use the strategy in `docs/technical/KMS_OIDC_STRATEGY.md` instead of treating HS256 as enterprise identity.

Spoofed actor accepted

Confirm AXIS derives actor, tenant, role, app, host, actor type, and environment from the verified token before building `RequestContext`.

Audit evidence missing

Check the `./enterprise-data:/app/data` mount, file ownership, disk space, and `AXIS_AUDIT_WAL_PATH` / `AXIS_AUDIT_LOG_PATH`.

Policy manifest invalid

Run the existing policy manifest update flow only after intentional policy review. Do not bypass manifest validation to start AXIS.
# AXIS Evidence Bundle V1

An AXIS Evidence Bundle is a portable JSON export of durable audit evidence from
`GET /audit/export`. It is intended for customer, auditor, and security review
handoff without requiring a running AXIS backend.

## Schema Version 1.0

Evidence Bundle V1 uses:

- `bundle_type`: `axis.evidence_bundle.v1`
- `schema.name`: `axis.evidence_bundle`
- `schema.version`: `1.0`
- `integrity.canonicalization`: `axis.canonical_json.v1`

Schema version `1.0` means the top-level object layout, integrity fields,
redaction contract, and verifier behavior are stable for V1 consumers. Future
breaking changes must use a new schema version or bundle type.

## Bundle Contents

The bundle includes:

- Bundle metadata: `bundle_id`, `generated_at`, generator identity, filters, and
  summary counts.
- Verification report from the AXIS audit verifier at export time.
- Active policy evidence: policy id, policy version, policy SHA-256, manifest
  loaded state, and policy integrity state.
- Exported audit events in a share-safe summary format.
- Integrity metadata: canonical payload hash, event-hash aggregate hash, optional
  Ed25519 signature metadata, and signing timestamp.
- Redaction metadata that states secrets, filesystem paths, and raw fields were
  excluded or redacted.

The bundle does not include raw audit records, signing private keys, operator
tokens, backend-only environment variables, or backend filesystem paths.

## What It Does Not Prove

An Evidence Bundle proves that the included AXIS evidence was exported in a
specific V1 structure and that the exported payload has not changed since the
recorded hash or signature was produced.

It does not prove external business truth. For example, it does not prove that a
third-party ticket, customer request, or database state outside AXIS was correct.
It also does not prove evidence that was not included in the export.

## Canonicalization

`axis.canonical_json.v1` is deterministic JSON used for hashing and signing:

- Object keys are ordered lexicographically.
- Arrays preserve their exported order.
- Null values are represented deterministically.
- Timestamps are RFC3339 strings.
- Whitespace is not part of the canonical payload.
- `integrity.payload_sha256` is excluded from the canonical payload because it is
  self-referential.
- `integrity.signature` is excluded from the canonical payload because the
  signature signs the payload, not itself.
- Other integrity metadata, including `signature_algorithm`,
  `signature_status`, `kid`, `signature_key_id`, and `signed_at`, is included
  in the payload hash and signature.

`event_hashes_sha256` is the SHA-256 of the canonical JSON array of included
non-empty `event_hash` values, in exported event order. If no event hashes are
included, it is `null`.

## Signing

Signing is optional and server-side only. AXIS supports Ed25519 signing when
configured with:

- `AXIS_EVIDENCE_SIGNING_ENABLED=true`
- `AXIS_EVIDENCE_SIGNING_KEY_ID=<key id>`
- `AXIS_EVIDENCE_SIGNING_PRIVATE_KEY_B64=<base64 raw private key>`
- `AXIS_EVIDENCE_SIGNING_PUBLIC_KEY_B64=<base64 raw public key>`

When signing is disabled, exports still work and are marked:

- `signature_algorithm`: `none`
- `signature_status`: `disabled`
- `kid`: `null`
- `signature_key_id`: `null`
- `signature`: `null`
- `signed_at`: `null`

Unsigned bundles are honest portable evidence, but they are not cryptographically
attributable to a signing key. Reviewers should treat them as hash-protected
exports and should obtain them through a trusted channel.

When signing is enabled and key material is valid, AXIS signs the canonical
unsigned payload with Ed25519 and records `signature_status: signed`, the key id
in `kid` and `signature_key_id`, and the signing timestamp. If signing is
enabled but key material is invalid, export fails safely with a
structured error and does not leak key material.

## Verification Scope

`verification.verification_scope` describes what AXIS checked at export time:

- `chain_continuity`: previous hash continuity only.
- `chain_continuity_and_event_hash`: previous hash continuity and event hash
  recomputation were checked.

The offline verifier checks the exported bundle structure, payload hash,
event-hash aggregate, and signature when applicable. It does not recompute AXIS
WAL event hashes from raw WAL records because raw records are not included.

## Redaction

Evidence Bundle V1 is designed for sharing. Exported events omit raw audit
records and raw SQL fields. Secret-like fields, private keys, operator tokens,
credentials, and backend filesystem paths are not included. The `redaction`
object records the redaction posture:

- `secrets_redacted: true`
- `filesystem_paths_redacted: true`
- `raw_fields_included: false`

## Verify a Bundle

Verify an unsigned or signed bundle structure and hashes:

```bash
python scripts/axis_evidence_bundle_verify.py --bundle axis-evidence-bundle-v1.json
```

Verify a signed bundle and require a valid signature:

```bash
python scripts/axis_evidence_bundle_verify.py --bundle axis-evidence-bundle-v1.json --public-key-b64 <key> --require-signature
```

Expected verifier outcomes:

- `AXIS Evidence Bundle Verification: PASS`
- `AXIS Evidence Bundle Verification: PASS_WITH_WARNINGS` with
  `reason: bundle_unsigned`
- `AXIS Evidence Bundle Verification: FAIL` with a reason such as
  `payload_sha256_mismatch`

The verifier returns a non-zero exit code on failure.

## Known Limitations

WAL reads are still linear. Large audit logs can make export and verification
latency proportional to the WAL size.

An Evidence Bundle proves included AXIS evidence, not external business truth.
# AXIS External Review Guide

## 1. AXIS Overview

AXIS is a database access guard that validates requests, classifies SQL, applies deterministic policy, records WAL-backed audit evidence, and exposes review APIs for audit, runtime metrics, policy status, approvals, and Evidence Bundle V1 export.

## 2. What AXIS Protects

- Production database write paths governed by policy.
- Approval-gated operations.
- Audit evidence continuity through a hash chain.
- Evidence export integrity through bundle hashing and optional Ed25519 signing.
- Runtime behavior under malformed requests, timeouts, rate limits, and pool pressure.

## 3. What AXIS Does Not Protect Yet

- Network perimeter, TLS termination, or identity provider integration.
- Full PostgreSQL protocol proxying.
- High-availability storage or backup orchestration.
- Indexed audit search at large scale.
- External business truth outside AXIS evidence.

## 4. Core Runtime Flow

HTTP request -> request validation -> SQL classification -> policy decision -> WAL audit evidence -> DB execution if allowed -> result evidence -> response -> optional audit verification or Evidence Bundle V1 export.

Protected writes require durable decision evidence before DB execution. AXIS does not silently repair or skip malformed audit records.

## 5. Policy Decision Flow

AXIS loads the active policy through `AXIS_POLICY_MANIFEST`, validates manifest integrity, runs startup dry-run checks, classifies the request, evaluates defaults and rules, and returns `ALLOW`, `BLOCK`, or `REQUIRE_APPROVAL`.

## 6. Approval Workflow

Approval-required decisions create immutable approval records. Resolution requires operator auth. Resolution produces audit evidence and should not allow double execution during concurrent resolve races.

## 7. Audit And Evidence Model

The WAL is canonical. The JSONL audit projection is operator convenience. Verification recomputes continuity and reports `verified`, `tampered`, `unverifiable`, or `error` honestly.

## 8. Evidence Bundle V1 Verification

Export:

```powershell
curl.exe "http://localhost:6543/audit/export?limit=10" -o axis-evidence-bundle-v1.json
python scripts/axis_evidence_bundle_verify.py --bundle axis-evidence-bundle-v1.json
```

Signed bundles can be verified with:

```powershell
python scripts/axis_evidence_bundle_verify.py --bundle axis-evidence-bundle-v1.json --public-key-b64 <public-key> --require-signature
```

## 9. Runtime Guardrails

Review:

- Request timeout.
- DB query and connect timeouts.
- DB pool acquisition timeout.
- Max request body and SQL sizes.
- Rate limiting.
- Startup fail-fast config validation.
- Structured errors without raw SQL, secrets, private keys, or filesystem paths.

## 10. Threat Model Summary

Primary threats are unsafe production writes, approval bypass, audit tampering, malformed request abuse, evidence export tampering, secret leakage, and operational failure under pressure. Current controls are deterministic policy, immutable approvals, WAL-backed evidence, hash-chain verification, redacted bundle export, optional signing, operator auth, timeouts, pool bounds, rate limiting, and Docker-backed validation.

## 11. Security Assumptions

- The backend environment is trusted to hold server-only secrets.
- PostgreSQL credentials and operator tokens are supplied by deployment tooling.
- Reviewers preserve audit volumes when evidence continuity matters.
- Browser code uses the Control Plane proxy and does not receive backend URLs or operator tokens.

## 12. Known Limitations

- Local Compose is not production HA.
- No built-in TLS or external identity provider integration.
- Audit explorer reads use a derived index for candidate selection when available. WAL remains canonical, index rebuild is synchronous today, and `/audit/verify` remains a full WAL verification scan.
- Evidence bundles do not include raw WAL records.
- Unsigned bundles are hash-protected but not attributable to a signing key.

## 13. How To Run The System

```powershell
docker compose down
docker compose up --build -d
docker compose ps
curl.exe http://localhost:6543/health
```

Production-like:

```powershell
Copy-Item .env.production.example .env.production.local
notepad .env.production.local
docker compose --env-file .env.production.local --profile production-like up --build -d postgres dbguard-production-like
curl.exe http://localhost:6545/health
```

## 14. How To Run Tests

```powershell
cargo fmt
cargo check
cargo test
python scripts/axis_audit_api_smoke.py --base http://localhost:6543
python scripts/axis_runtime_smoke.py --base http://localhost:6543
python axis_regression.py --base http://localhost:6543
python axis_audit_restart_test.py --base http://localhost:6543
python axis_chaos_test.py --base http://localhost:6543
```

Frontend:

```powershell
cd control-plane
npm run typecheck
npm run lint
npm run build
npm run smoke:real
npx playwright test e2e/real-mode.spec.ts
```

## 15. How To Run Stress Validation

```powershell
python scripts/axis_runtime_stress.py --base http://localhost:6543 --concurrency 25 --requests 500 --include-export --include-rate-limit
```

Expected result:

```text
AXIS Runtime Stress Validation: PASS
```

A failure means the runtime produced an unsafe decision, unstructured error, unverifiable audit chain, broken evidence bundle, or leaked forbidden text.

## 16. What Reviewers Should Inspect

- `src/gate/listener.rs`
- `src/gate/enforcer.rs`
- `src/audit/logger.rs`
- `src/audit/reader.rs`
- `src/audit/verifier.rs`
- `src/audit/evidence_bundle.rs`
- `src/config.rs`
- `src/db/postgres.rs`
- `policies/policy_manifest.json`
- `policies/prod_main.json`
- `control-plane/src/app/api/axis/[...path]/route.ts`
- `Dockerfile`
- `docker-compose.yml`
- `.env.example`
- `.env.production.example`

## 17. Expected Review Questions

- What happens if audit WAL append fails before a protected write?
- Can a malformed audit record be skipped or repaired silently?
- Can an approval be resolved twice under concurrency?
- Are dangerous writes ever allowed under DB or audit failure?
- Does the browser receive `AXIS_OPERATOR_TOKEN`, signing keys, or backend filesystem paths?
- Are Evidence Bundle V1 hashes and signatures verifiable offline?
- Does startup fail on missing or mismatched policy manifests?
- Are stress and chaos results reproducible from documented commands?
# AXIS KMS and OIDC Strategy

This document describes the intended production direction. It does not claim that AXIS currently implements full KMS, Vault, HSM, OIDC, or JWKS network integration.

## Current State

AXIS supports local/demo JWT HS256 authentication for authenticated context validation. HS256 is useful for deterministic local and reviewer checks because it proves that AXIS derives actor, tenant, role, app, host, actor type, and environment from a verified bearer token instead of trusting request JSON fields.

HS256 is not full enterprise IAM.

Evidence bundle signing supports local Ed25519 key material and now reports signature metadata honestly:

- `signature_status`
- `signature_algorithm`
- `kid` / `key_id`
- `signed_at` when signed

Unsigned exports are marked disabled/unsigned and are not presented as signed evidence.

## Production Identity Direction

Production identity should use one or more trusted identity mechanisms:

- OIDC/JWKS validation against an enterprise identity provider
- mTLS with certificate identity mapping
- cloud IAM or workload identity
- service identity from the deployment platform
- short-lived credentials issued by an approved control plane

The future OIDC/JWKS adapter should validate issuer, audience, expiry, signature, key id, and claims mapping without trusting body-supplied identity fields.

## Production Signing Key Direction

Production evidence signing keys should use managed key infrastructure where possible:

- cloud KMS
- HSM
- managed secret store with audited access
- Vault-backed signing adapter
- hardware-backed or service-backed signing where available

Private signing material should not be committed, copied into container images, returned through APIs, or written into reports.

## Adapter Direction

Future adapters should keep the current honest metadata model:

- report `signature_status: disabled` when signing is not configured
- report `signature_status: signed` only after a real signature is produced
- report `signature_status: ephemeral_demo` only for explicitly ephemeral demo signing
- include `kid` for key identity and rotation review
- fail safely when signing is required but key material or remote signing is unavailable

This hardening work prepares the metadata and redaction boundaries. It does not implement automatic rotation, external key store connectivity, or network identity-provider validation.
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
# AXIS Limitations

## Review/demo scope

AXIS reviewer demo is intended for technical evaluation and pilot-readiness validation. It is not a full production deployment guide.

## PostgreSQL scope

AXIS is currently focused on PostgreSQL-oriented database control paths.

## Native wire protocol scope

If the AXIS reviewer demo uses HTTP gate mode, HTTP demo mode and native PostgreSQL wire protocol support are separate concerns. Passing the reviewer HTTP demo does not prove native PostgreSQL wire protocol coverage.

## Direct database bypass

AXIS cannot protect database traffic that bypasses AXIS entirely. Production deployments must enforce network, IAM, firewall, security group, and database access controls so protected write paths must pass through AXIS.

The enterprise compose profile demonstrates this boundary by keeping PostgreSQL off host port 5432 and by using a read-only proof role, but customer production environments must enforce the same boundary with their own network and credential controls.

## Identity boundary

JWT HS256 enterprise demo mode is not a complete enterprise IAM solution. It proves that AXIS can derive policy and audit context from a verified token instead of trusting spoofable request JSON fields. Production deployments should integrate with OIDC/JWKS, mTLS, cloud IAM, service identity, or equivalent trusted identity infrastructure.

HS256 mode remains local/demo authentication and is not full enterprise IAM. Local secrets reduce accidental leak risk for validation runs only; they do not provide managed production identity, rotation, revocation, or centralized access audit.

## Local manifest integrity

SHA-256 manifest validation proves local file integrity. It is not the same as asymmetric signing, remote attestation, or hardware-backed trust.

## Key management

Local signing and authentication secrets are not equivalent to managed KMS or HSM-backed key management. Production secret material should be generated, stored, rotated, and audited through managed key infrastructure.

AXIS does not yet provide native KMS, HSM, Vault, or cloud key store integration. Evidence signing metadata reports whether signing is disabled or signed, and which key id was used when signing is configured, but key lifecycle remains an operator responsibility.

Memory zeroing reduces accidental retention after a secret wrapper is dropped. It does not make live process memory impossible to inspect. Production deployments should disable core dumps and use managed secret infrastructure.

## High availability

A single-node AXIS deployment is not a full high availability architecture. Production designs need explicit availability, failover, observability, and recovery planning for AXIS, PostgreSQL connectivity, audit storage, and policy distribution.

## Audit durability assumptions

AXIS can generate durable audit evidence, but real production durability also depends on:

- disk reliability
- volume configuration
- backup strategy
- retention policy
- monitoring
- access controls around evidence storage

## ORM-generated SQL

AXIS uses deterministic SQL parsing and policy evaluation. ORMs such as Prisma, Hibernate, Entity Framework, Sequelize, and similar tools can generate deeply nested, complex, vendor-specific, or unusual SQL shapes.

Early pilot deployments are optimized for explicit, reviewable, and predictable SQL patterns.

Extremely complex ORM-generated query shapes may be rejected or fail-closed until they are profiled, classified, and added to the supported policy and regression corpus.

This behavior is intentional. AXIS prefers fail-closed behavior over silently allowing SQL it cannot classify with confidence.

## Performance benchmark limitation

Reviewer demo benchmark results are not a substitute for customer-specific workload testing.

## Pilot deployment boundary

Production pilot deployments should start with a narrow protected write path, defined policy scope, known database schema, and agreed rollback/incident procedure.
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# AXIS Native PostgreSQL Integration RFC v1.2

## Purpose

This RFC set defines the architecture, risk boundaries, implementation constraints, and production hardening requirements for running VARUX AXIS as a Native PostgreSQL security enforcement proxy.

This is not a user guide and it is not a marketing packet. It is the architectural source of truth before implementation. The purpose is to prevent AXIS from becoming a fragile generic database proxy while still enabling it to sit in the PostgreSQL traffic path where real production applications live.

AXIS remains a deterministic data security enforcement point. Native PostgreSQL support is an integration mode, not the product identity.

## Why v1.2 Exists

v1.1 made the architecture more honest, but external review identified remaining production blockers:

- Audit WAL evidence format was referenced but not specified.
- CancelRequest was marked as required but not designed.
- Performance targets were not tied to audit durability choices.
- Approval store high availability was underspecified.
- Emergency bypass created an evidence gap.
- Observability lacked on-call workflows and high-cardinality guardrails.
- Extended Query remained acknowledged but still insufficiently close to the critical path.
- Multi-AXIS consistency, policy version drift, backend failover, and restart recovery required sharper operational semantics.

v1.2 is therefore a hardening addendum and consolidation pass.

## Product Boundary

AXIS Native PG mode is a security enforcement proxy. It accepts PostgreSQL wire traffic, inspects the SQL-bearing portions of that traffic, evaluates policy before execution, blocks or routes dangerous operations, and emits durable evidence.

AXIS is not:

- a PostgreSQL connection pooler;
- a general query accelerator;
- a transparent replacement for PgBouncer or RDS Proxy;
- a WAF;
- a SQL injection detector;
- a database migration framework;
- a guarantee that ALLOW means safe.

AXIS ALLOW means only this: the request satisfied the configured AXIS policy under the context AXIS observed. It does not mean the application input was safe, the query was semantically harmless, or the business operation was correct. Humans do love turning permission into moral approval. AXIS will not participate in that little tragedy.

## v1.2 Document Map

| File | Purpose |
|---|---|
| 00-README.md | Entry point, scope, document map, release posture |
| 01-STRATEGIC-RATIONALE.md | Why Native PG exists and why HTTP/query still remains |
| 02-DEPLOYMENT-TOPOLOGIES.md | Supported and forbidden deployment shapes |
| 03-PROTOCOL-BOUNDARIES.md | PostgreSQL protocol phases and boundaries |
| 04-TRANSACTION-STATE-MODEL.md | Transaction handling, poisoning, rollback, strict/lenient posture |
| 05-AUTH-TLS-STRATEGY.md | TLS, SCRAM, pass-through limitations, future mTLS |
| 06-POLICY-AUDIT-MAPPING.md | Mapping wire requests into policy and evidence |
| 07-POC-SCOPE.md | Lab-only Simple Query PoC scope |
| 08-FAILURE-MODES.md | Failure behavior and fail-safe semantics |
| 09-TEST-MATRIX.md | Test requirements, including byte-level backend mock |
| 10-CODEX-IMPLEMENTATION-BRIEF.md | Implementation brief with module boundaries |
| 11-PROTOCOL-FIDELITY-MATRIX.md | Feature support matrix across protocol behaviors |
| 12-IDENTITY-ATTRIBUTION-STRATEGY.md | Actor identity, backend attribution, protected metadata |
| 13-THREAT-MODEL-AND-BYPASSES.md | Parser, protocol, semantic, side-channel, DoS threats |
| 14-PERFORMANCE-BUDGET.md | Latency and throughput targets |
| 15-APPLICATION-INTEGRATION-GUIDE.md | Driver and application integration expectations |
| 16-RISK-REGISTER.md | Consolidated risk register |
| 17-RESTART-RECOVERY-SEMANTICS.md | AXIS crash/restart and in-flight uncertainty |
| 18-APPROVAL-IDEMPOTENCY-MODEL.md | Approval ticket scope, replay, deduplication |
| 19-EMERGENCY-BYPASS-PROCEDURE.md | Operational bypass and forensic procedure |
| 20-OPERABILITY-OBSERVABILITY.md | Metrics, traces, logs, dashboards, runbooks |
| 21-RFC-LIFECYCLE-AND-VERSIONING.md | RFC ownership and evolution rules |
| 22-EXTENDED-QUERY-ROADMAP.md | Parse/Bind/Execute roadmap and production gate |
| 23-CLUSTER-FAILOVER-AND-MULTI-AXIS.md | Multi-instance, failover, policy consistency |
| 24-POLICY-AUTHORING-FOR-NATIVE-PG.md | Policy writing and validation for Native PG mode |
| 25-AUDIT-WAL-FORMAT-AND-EVIDENCE-SPEC.md | WAL entry schema, hash chain, export, verification |
| 26-CANCELREQUEST-DESIGN.md | CancelRequest key mapping and proxy design |
| 27-PERFORMANCE-AND-DURABILITY-TRADEOFFS.md | fsync, group commit, crash window, durability modes |
| 28-APPROVAL-STORE-HA-AND-CONSISTENCY.md | Approval store replication, races, HA posture |
| 29-BYPASS-AUDIT-GAP-AND-RECONCILIATION.md | Audit-only mode and bypass reconciliation |

## Current Implementation Posture

| Capability | Status |
|---|---|
| HTTP/query policy enforcement | Existing AXIS core |
| Audit WAL/hash-chain core | Existing AXIS core |
| Native PG Simple Query PoC | Planned lab-only implementation |
| Extended Query support | Required before real OLTP pilot |
| CancelRequest support | Required before real OLTP pilot |
| Production TLS/mTLS | Required before enterprise production |
| Approval store HA | Required before multi-AXIS pilot |
| Audit WAL evidence spec | Added in v1.2 |
| Emergency bypass reconciliation | Added in v1.2 |

## Design Principles

1. Do not forward protected write operations without policy decision and durable evidence intent.
2. Do not pretend to know backend state when backend state is unknown.
3. Do not let approval flows hold TCP sockets hostage.
4. Do not conflate AXIS audit identity with PostgreSQL authenticated identity unless AXIS actually verified it.
5. Do not rewrite user SQL except for explicitly documented metadata correlation mechanisms.
6. Do not support PostgreSQL features partially and call them supported.
7. Do not let performance targets erase evidence guarantees.
8. Do not let emergency bypass silently destroy the audit story.

## Production Gate Summary

A real production pilot must not start until:

- Extended Query Parse/Bind/Execute support is implemented and tested for the target driver stack.
- CancelRequest is implemented or explicitly disabled with customer acceptance.
- Audit WAL format and verification are stable.
- Approval store HA posture is defined.
- Identity attribution model is accepted by the customer.
- PgBouncer mode is validated and transaction pooling is disabled.
- Performance and durability mode are chosen.
- Emergency bypass and reconciliation procedure are rehearsed.
- Observability dashboard and on-call runbook exist.

## Success Looks Like

An engineer can read this RFC set and know exactly what AXIS will do before forwarding, after forwarding, during failure, during approval, during restart, and during bypass.

## Failure Looks Like

A developer reads these files and still writes a monolithic `proxy.rs` that pretends Simple Query support equals production PostgreSQL compatibility. That file belongs in a museum of avoidable mistakes.
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---


# Strategic Rationale

## Purpose

This document explains why AXIS needs native PostgreSQL integration, why HTTP/query mode remains valuable, and why native mode must be treated as a Security Enforcement Proxy rather than a generic database proxy.

## The Existing AXIS Value

AXIS already provides deterministic controls around SQL operations:

- SQL classification before execution.
- Policy decisioning.
- ALLOW / BLOCK / APPROVAL_REQUIRED outcomes.
- Approval workflow.
- Durable audit/evidence.
- Policy manifest integrity.
- Fail-safe behavior.

That value is not the proxy. The value is enforcement before the database is touched.

## Why HTTP/query Is Not Enough

HTTP/query mode is useful where the caller is intentionally integrated with AXIS: internal tools, migration pipelines, DBA workflows, CI/CD controls, scripted maintenance, and demos.

It is not enough for transparent OLTP traffic because most production applications already speak PostgreSQL through drivers, ORMs, and connection pools. Asking every customer to rewrite database access into an AXIS HTTP API is an adoption barrier large enough to kill enterprise deployment.

## Why Native PG Mode Is Needed

Native PG mode lets AXIS sit in the PostgreSQL traffic path:

```text
Application / Driver / ORM -> AXIS -> PostgreSQL
```

The goal is not “no change ever.” The truthful claim is:

> AXIS integrates into PostgreSQL wire traffic with connection string changes and verified driver compatibility. Approval workflows may require application-level error handling.

This is less shiny than “zero app change,” but it survives reviewer questions. Strange how honesty works better than marketing confetti.

## What AXIS Is

AXIS Native PG mode is:

- A Security Enforcement Proxy.
- A Policy Enforcement Point.
- An audit evidence producer.
- A controlled transport integration layer.
- A deterministic data security enforcement point.

## What AXIS Is Not

AXIS is not:

- A PgBouncer replacement.
- A PostgreSQL performance proxy.
- A load balancer.
- A general database gateway.
- A query optimizer.
- A WAF.
- A replacement for application input validation.
- A full PostgreSQL server implementation.

## Product Boundary

AXIS inspects database operations to enforce policy and produce evidence. It may forward PostgreSQL protocol messages, but forwarding is in service of enforcement, not the product identity.

If transport complexity starts consuming policy, audit, and fail-safe quality, the native mode has become a distraction.

## Strategic Risk

Native PG mode moves AXIS toward Tier 0 infrastructure. If AXIS fails closed too aggressively, customer applications may degrade. If AXIS fails open, the product loses its reason to exist. If AXIS silently diverges from backend state, audit claims become dangerous.

## v1.2 Position

Native PG integration is strategically correct, but must be staged:

1. RFC and risk model.
2. Simple Query lab PoC.
3. Extended Query state machine.
4. CancelRequest mapping.
5. Identity verification and attribution.
6. Observability and emergency bypass.
7. Controlled pilot.
8. Production readiness review.

## Current Known Weaknesses

- Native mode dramatically expands the maintenance burden.
- Simple Query proves transport viability, not product compatibility.
- Enterprise value depends on Extended Query, identity, observability, and failure recovery.
- Calling AXIS “not a proxy” is inaccurate in native mode. The right phrase is “security enforcement proxy, not generic database proxy.”

## Acceptance Criteria

This rationale is accepted only if future implementation work preserves AXIS as an enforcement product rather than turning it into a transport science project with audit stickers glued on.
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---


# Deployment Topologies

## Purpose

This document defines where AXIS may sit in relation to applications, PgBouncer, PostgreSQL, sidecars, gateways, and multi-instance deployments.

## Topology A: Lab PoC

```text
psql / test client -> AXIS -> PostgreSQL
```

### Use

Simple Query PoC only.

### Benefits

- Minimal moving parts.
- Clear state ownership.
- Easier byte-level tests.

### Limitations

- Not representative of production ORMs.
- No pooling.
- No HA.
- No TLS/mTLS.

## Topology B: Application to AXIS to PostgreSQL

```text
Application -> AXIS -> PostgreSQL
```

### Use

Early controlled application tests.

### Benefits

- AXIS sees the application connection directly.
- Session state is easier to reason about.
- Identity and application_name are visible before pooler modification.

### Risks

- AXIS must handle connection count.
- Scaling pressure arrives early.
- No external pooling help.

## Topology C: Application to AXIS to PgBouncer to PostgreSQL

```text
Application -> AXIS -> PgBouncer(session pooling) -> PostgreSQL
```

### Use

Likely first realistic pilot topology if pooling is required.

### Benefits

- AXIS sees client-side intent before PgBouncer.
- PgBouncer handles backend connection pressure.
- AXIS does not become a pooler.

### Requirements

- PgBouncer must use session pooling unless proven otherwise.
- Transaction pooling is not supported for initial native mode.
- AXIS must retain clear transaction state from client traffic.
- PgBouncer behavior must be included in the test matrix.

## Topology D: Application to PgBouncer to AXIS to PostgreSQL

```text
Application -> PgBouncer -> AXIS -> PostgreSQL
```

### Status

Not recommended for early pilots.

### Risk

PgBouncer may hide per-client state, collapse identity, multiplex transactions, and make AXIS believe a stable session exists where one does not. This is how architecture starts lying politely.

## Topology E: Kubernetes Sidecar

```text
App container -> localhost:5432 -> AXIS sidecar -> PostgreSQL service
```

### Use

Strong candidate for controlled Kubernetes pilots.

### Benefits

- Low network distance.
- Per-workload blast radius.
- Clear deployment ownership.
- Easier per-app policy binding.

### Requirements

- Persistent or remote audit WAL strategy.
- Sidecar health checks tied to audit and policy readiness.
- Graceful drain during pod shutdown.
- Emergency bypass procedure.

## Topology F: Central Gateway

```text
Applications -> AXIS gateway cluster -> PostgreSQL
```

### Use

Later enterprise deployment.

### Benefits

- Centralized policy enforcement.
- Central observability.
- Easier operational control.

### Risks

- Tier 0 blast radius.
- Requires HA, routing, failover, global policy consistency, and approval deduplication.
- Requires multi-AXIS audit chain strategy.

## PgBouncer Compatibility Decision

| PgBouncer Mode | Status | Reason |
|---|---|---|
| Session pooling | Candidate | Preserves session continuity better |
| Transaction pooling | Unsupported initially | Breaks AXIS transaction/session assumptions |
| Statement pooling | Unsupported | Incompatible with stateful enforcement |

## Multi-AXIS Consistency

Multiple AXIS instances introduce consistency questions:

- Are all instances on the same policy version?
- Is approval ticket dedup global?
- Are audit chains per-instance or globally anchored?
- How are `axis_request_id` and `approval_ticket_id` generated?
- What happens during rolling upgrades when policy versions differ?

The initial answer:

- Per-instance audit WAL is allowed.
- Global evidence export must include instance ID, policy version, and wall-clock time.
- Approval ticket store must be shared or single-writer before multi-instance approval support.
- Policy rollout must use manifest version pinning.
- Mixed policy versions must be visible and considered degraded.

## Current Known Weaknesses

- No production topology is safe until Extended Query, CancelRequest, observability, emergency bypass, and restart recovery semantics exist.
- Session pooling assumptions must be tested, not merely admired.
- Multi-instance AXIS requires coordination that v1.0 did not need.

## Acceptance Criteria

A topology is acceptable only if AXIS can answer: which client sent the query, which policy version evaluated it, which backend received it, what state the transaction was in, what audit evidence proves the result, and how the operator bypasses AXIS if it misbehaves.
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---


# PostgreSQL Protocol Boundaries

## Purpose

This document defines the PostgreSQL wire protocol boundaries for AXIS Native PG mode.

## Core Principle

AXIS must only claim support for protocol behavior it can safely observe, classify, forward, block, and audit.

Unsupported does not mean “probably fine.” Unsupported means blocked, rejected, or explicitly excluded from deployment.

## Startup Phase

### PoC Behavior

For the lab PoC, AXIS proxies startup/authentication messages between client and backend as transparently as possible.

### Important Limitation

In pass-through auth mode, AXIS does not verify the client identity. `claimed_db_user` is only the user string observed in StartupMessage and relayed through backend authentication. This must not be represented as AXIS-verified identity.

### Messages

AXIS must explicitly handle or pass through:

- StartupMessage.
- SSLRequest.
- Authentication messages.
- PasswordMessage / SASL messages.
- ParameterStatus.
- BackendKeyData.
- ReadyForQuery.
- NegotiateProtocolVersion.

## Simple Query

Simple Query is message type `Q`.

PoC supports Simple Query only for enforcement:

1. Client sends Query.
2. AXIS extracts SQL.
3. AXIS builds AxisRequestEnvelope.
4. Policy evaluates.
5. ALLOW forwards original bytes.
6. BLOCK or APPROVAL_REQUIRED does not forward query bytes.
7. AXIS returns a PostgreSQL-compatible ErrorResponse and ReadyForQuery when appropriate.

## Original Bytes Rule

For user SQL, ALLOW path must forward original query bytes. AXIS must not rewrite the SQL statement to add comments, hints, correlation IDs, or metadata.

### Correlation Exception

AXIS may issue separate session metadata commands such as `SET application_name` or safe custom GUC setup before user queries, but those are AXIS-generated control statements and must be audit-recorded separately. AXIS must not silently rewrite the user query.

## Extended Query

Extended Query includes Parse, Bind, Describe, Execute, Sync, Close, and related state.

Status: not supported in Simple Query PoC. Required before production OLTP pilot.

Reason: modern drivers and ORMs commonly use Extended Query. Supporting only Simple Query proves transport viability, not production compatibility. There, we admitted it, because denial is cheaper only until production.

## COPY

COPY must be divided into two threat classes:

1. SQL-level COPY statements, including `COPY FROM PROGRAM` and `COPY TO PROGRAM`.
2. Protocol-level COPY streams, including CopyData, CopyDone, CopyFail.

PoC behavior:

- Any COPY statement is blocked.
- `COPY FROM PROGRAM` and `COPY TO PROGRAM` are classified as critical dangerous operations.
- Any protocol COPY stream state is unsupported and must be rejected or connection-closed safely.

## CancelRequest

CancelRequest is out-of-band and uses a new connection.

PoC may mark CancelRequest unsupported, but production pilot must support AXIS-generated cancel keys and backend key mapping. Passing backend BackendKeyData directly to clients creates cancellation bypass and DoS risk.

## ReadyForQuery

ReadyForQuery contains transaction status:

- `I`: idle.
- `T`: in transaction.
- `E`: failed transaction.

AXIS must track this status from backend messages. If AXIS generates ReadyForQuery after a local BLOCK outside transaction, it must use a status consistent with AXIS/backend state.

If AXIS cannot guarantee consistency, it must close the connection and audit why.

## ErrorResponse

AXIS-generated ErrorResponse must define fields exactly:

- `S`: ERROR or FATAL.
- `V`: ERROR or FATAL.
- `C`: exact SQLSTATE.
- `M`: human-readable message.
- `D`: structured detail including `axis_request_id` and optionally `axis_ticket_id`.
- `H`: safe hint.
- Avoid misleading schema/table/column fields unless known and safe.

No “42501-style” language. Pick exact codes.

Initial recommendation:

- BLOCK: `42501` insufficient_privilege.
- APPROVAL_REQUIRED: vendor-defined structured error using `P0001` or `42501` with machine-readable Detail. Final choice must be tested against drivers.

## CommandComplete

AXIS must pass through CommandComplete but also extract row count for audit when possible:

- `INSERT 0 n`
- `UPDATE n`
- `DELETE n`
- `SELECT n`

Row count is part of enforcement evidence.

## NoticeResponse and Async Messages

NoticeResponse is non-fatal and should be passed through unless it contains unsupported protocol state.

LISTEN/NOTIFY is not supported in PoC and must be deployment-excluded.

## Pipeline Mode

Pipeline mode is unsupported. If detected or suspected, AXIS must fail closed. Request-response assumptions do not hold under pipelining.

## Large Query Handling

AXIS must enforce query size limits before unbounded buffering. A client must not be able to force AXIS to allocate a 50MB query before rejection.

## Current Known Weaknesses

- PoC pass-through auth does not verify identity.
- Simple Query-only does not cover most modern production ORM traffic.
- CancelRequest remains a production blocker until mapped.
- Protocol features must be tested at byte level, not inferred from PostgreSQL logs.

## Acceptance Criteria

A protocol feature is considered supported only if AXIS has a defined behavior for parsing, state impact, forwarding/blocking, audit evidence, and failure handling.
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# Transaction State Model

## Purpose

This document defines how AXIS tracks, constrains, and recovers PostgreSQL transaction state in Native PG mode.

Transaction behavior is the largest correctness risk in AXIS Native PG mode. If client, AXIS, and backend PostgreSQL disagree about transaction state, enforcement and audit claims become unreliable.

## Scope

This document covers:

- transaction control command classification;
- strict and future lenient transaction behavior;
- BLOCK and APPROVAL_REQUIRED inside and outside transactions;
- AXIS-issued safety ROLLBACK behavior;
- ReadyForQuery transaction status handling;
- connection poisoning;
- commands such as DISCARD ALL, RESET ALL, SAVEPOINT, and client ROLLBACK;
- retry storm risks and application expectations.

## Non-Goals

- Implementing full savepoint continuation in the Simple Query PoC.
- Guaranteeing zero application disruption for blocked in-transaction writes.
- Providing a distributed transaction coordinator.
- Making dangerous transactions pleasant. Security controls are rarely spa treatments.

## Core Risk: Transaction Divergence

Divergence means the parties disagree:

| Party | Example Belief |
|---|---|
| Client | Transaction is active and can be committed |
| AXIS | Transaction was poisoned by a blocked statement |
| Backend | Transaction is active with only some statements executed |

This is not cosmetic. A blocked statement inside a transaction may leave earlier allowed statements pending. If the client later commits, those earlier statements become durable even though the transaction contained a policy violation.

## State Sources

AXIS must combine:

- SQL classification of transaction commands;
- backend `ReadyForQuery` transaction status byte;
- internal session state;
- audit intent/completion records;
- failure events;
- connection lifecycle events.

Backend `ReadyForQuery` remains the strongest source for backend transaction state after forwarded queries. AXIS-generated `ReadyForQuery` must be used only when backend was not touched or after AXIS has explicitly reset/closed the session.

## Transaction States

| State | Meaning | Reuse Allowed |
|---|---|---|
| Idle | No active transaction | Yes |
| InTransaction | Active backend transaction | Conditional |
| FailedTransaction | Backend transaction error state | Only rollback/reset |
| PoisonedByAXIS | AXIS blocked or interrupted operation and cannot safely continue | No |
| ResetAttempted | AXIS issued safety ROLLBACK/reset | No until confirmed |
| ResetConfirmed | Backend confirmed rollback/reset | Future optional reuse |
| ExecutionUnknown | Backend execution status unknown | No |
| Closed | Connection closed | No |

## ReadyForQuery Status Byte Rules

PostgreSQL `ReadyForQuery` carries transaction status:

| Byte | Meaning |
|---|---|
| `I` | Idle |
| `T` | In transaction |
| `E` | Failed transaction |

AXIS must never emit a misleading status byte.

### BLOCK outside transaction

Backend is Idle and query is not forwarded:

- AXIS may emit ErrorResponse followed by ReadyForQuery `I`.
- Connection may remain open.

### APPROVAL_REQUIRED outside transaction

Backend is Idle and query is not forwarded:

- AXIS may emit ErrorResponse with structured approval detail followed by ReadyForQuery `I`.
- Connection may remain open.

### BLOCK inside transaction

AXIS must not emit `I` unless backend reset was confirmed. v1.2 strict default is:

1. Do not forward blocked query.
2. Return ErrorResponse if possible.
3. Issue policy-exempt safety ROLLBACK to backend.
4. Audit rollback attempt and result.
5. Close client connection.

If rollback is confirmed before close, audit records `ResetConfirmed`. If rollback is not confirmed, audit records `ExecutionUnknown` or `TransactionResetUnknown` depending on dispatch state.

### APPROVAL_REQUIRED inside transaction

Approval cannot wait while locks remain active. v1.2 strict default:

1. Create or reuse approval ticket.
2. Do not forward original query.
3. Issue safety ROLLBACK.
4. Return ErrorResponse if possible.
5. Close connection.
6. Audit `approval_required_in_transaction=true` and `transaction_reset_attempted=true`.

## Policy Treatment of Transaction Control Commands

| Command | v1.2 Handling | Notes |
|---|---|---|
| BEGIN | Policy-evaluated, usually allow | Starts AXIS transaction tracking |
| START TRANSACTION | Same as BEGIN | Must normalize synonyms |
| COMMIT | Policy-evaluated unless session poisoned | Poisoned sessions must not commit |
| END | Same as COMMIT | Alias |
| Client ROLLBACK | Always allowed unless protocol invalid | Must reset AXIS state after backend confirms |
| AXIS safety ROLLBACK | Policy-exempt, audit-recorded | Safety primitive, not user operation |
| SAVEPOINT | Unsupported in Simple Query PoC or policy-blocked | Required for future lenient mode |
| ROLLBACK TO SAVEPOINT | Unsupported in Simple Query PoC or policy-blocked | Future lenient support |
| RELEASE SAVEPOINT | Unsupported in Simple Query PoC or policy-blocked | Future lenient support |
| DISCARD ALL | Block by default | Can reset state outside AXIS expectations |
| RESET ALL | Block or evaluate against protected GUC list | May clear identity/correlation context |
| SET | Evaluated; protected GUCs blocked | Includes search_path, role, timeout, AXIS metadata |
| SET ROLE | Block by default | Identity ambiguity |
| SET SESSION AUTHORIZATION | Block by default | Identity bypass risk |

## Safety ROLLBACK Exception

When AXIS issues ROLLBACK to recover from BLOCK or APPROVAL_REQUIRED inside an active transaction, that ROLLBACK bypasses normal policy evaluation.

This exception must be explicit, documented, and auditable:

```json
{
  "event_type": "AXIS_SAFETY_ROLLBACK_ISSUED",
  "policy_exempt": true,
  "reason": "transaction_poisoned_by_block_or_approval",
  "axis_request_id": "req_...",
  "session_id": "sess_...",
  "backend_session_id": "backend_...",
  "original_decision": "BLOCK",
  "rollback_dispatch_intent": true
}
```

Rollback response must produce one of:

| Result | Meaning |
|---|---|
| `rollback_confirmed` | Backend acknowledged reset |
| `rollback_failed` | Backend returned error |
| `rollback_unknown` | Connection failed before confirmation |
| `rollback_not_attempted` | Backend was unreachable or state unknown |

## Strict Mode

Strict mode is the v1.2 default.

Behavior:

- Outside transaction: BLOCK/APPROVAL_REQUIRED rejects only the statement and may keep connection open.
- Inside transaction: BLOCK/APPROVAL_REQUIRED poisons the session, attempts rollback, audits, and closes.
- Unsupported protocol inside transaction poisons and closes.
- Execution unknown closes.

Why strict is default:

- It avoids false confidence about transaction state.
- It prevents COMMIT after a policy violation.
- It is easier to audit.
- It is harsh but coherent. Coherence wins.

## Lenient Mode: Future Experimental Design

Lenient mode is not part of Simple Query PoC and must not be enabled in production until specifically implemented and tested.

A future lenient mode may use AXIS-managed savepoints:

1. Before a risky candidate statement, AXIS creates a savepoint.
2. If policy BLOCKs, AXIS rolls back to that savepoint.
3. Transaction may continue.
4. Audit marks `partial_transaction_continued=true`.
5. Policy must explicitly allow lenient continuation.

Hard requirements:

- AXIS must own savepoint names and prevent client collision.
- Savepoint commands must be hidden from client or reconciled correctly.
- Backend and client ReadyForQuery semantics must remain valid.
- Driver behavior must be tested.
- `COMMIT` after blocked statement must be policy-controlled.
- Audit must record preceding statements hash and continuation risk.

Known concern: lenient mode can create a dangerous illusion that a partially blocked business transaction is still semantically valid. Databases cannot infer business meaning. Amazing that this has to be said, but here we are.

## Retry Storm Risk

Strict rollback+close may trigger application retry storms. Mitigation:

- ErrorResponse must carry machine-readable AXIS reason.
- Application integration guide must recommend bounded retries.
- AXIS may rate-limit repeated risky queries.
- Approval tickets must deduplicate repeated attempts.
- Connection pools must discard poisoned connections.
- Operators must observe `transaction_poisoned_count` and `retry_after_block_count`.

## Application Contract

Applications must treat the following as non-retryable without human/operator logic:

- AXIS policy BLOCK.
- AXIS approval required.
- AXIS transaction poisoned.
- AXIS execution unknown.

Applications may retry only after:

- approval was granted;
- transaction was rebuilt from the beginning;
- the operation is idempotent;
- business logic permits retry.

## Current Known Weaknesses

- Strict mode may be operationally disruptive for long-lived transactions.
- Lenient mode is not yet implemented.
- Savepoint semantics are complex and driver-dependent.
- Rollback confirmation may be lost during crash.
- Transaction boundaries hidden behind PgBouncer transaction pooling are unsupported.

## Success Looks Like

AXIS never allows a client to commit a transaction after AXIS knows that transaction was poisoned by a blocked or approval-required operation.

## Failure Looks Like

AXIS blocks a dangerous statement, leaves earlier statements pending, allows COMMIT, and then writes a confident audit event as if nothing weird happened. That would be security theater with a Rust compiler.

## Acceptance Criteria

- Every transaction state transition is auditable.
- Safety ROLLBACK is explicitly policy-exempt and logged.
- BLOCK/APPROVAL inside transaction cannot silently continue.
- Client-visible ErrorResponse behavior is deterministic.
- Connection pool discard guidance exists.
- Retry storm metrics exist.
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---


# Auth and TLS Strategy

## Purpose

This document defines authentication and TLS strategy for AXIS Native PG mode.

## PoC Mode

PoC mode may use cleartext local TCP and startup/auth pass-through.

This is acceptable only for lab work.

## Critical PoC Limitation

In auth pass-through mode, AXIS does not verify the client's identity. It relays the backend authentication exchange. The observed database user is `unauthenticated_claimed_db_user` from AXIS's perspective.

This must be stated in audit and documentation.

## TLS Modes

### TLS Pass-Through

AXIS forwards encrypted bytes and cannot inspect SQL.

Status: not useful for policy enforcement.

### TLS Termination at AXIS

Client establishes TLS to AXIS. AXIS decrypts traffic, enforces policy, then opens a separate TLS/plain connection to backend depending on deployment.

Status: required direction for real enforcement.

### mTLS Termination

Client authenticates to AXIS with certificates. AXIS maps cert identity to actor/service identity and opens backend connection with controlled credentials.

Status: preferred enterprise target.

## SCRAM

SCRAM pass-through is complex and weak for AXIS identity assurance.

If AXIS simply relays SCRAM, backend authenticates the client but AXIS still cannot claim it independently verified identity unless it validates the exchange or trusts backend response as identity proof.

Enterprise target should evaluate:

- AXIS-managed auth.
- mTLS.
- OIDC/LDAP integration.
- Service account backend access.
- Per-actor backend auth only where operationally feasible.

## Backend Service Account

Service account backend access is acceptable for PoC and may be acceptable in enterprise if identity attribution is preserved via AXIS audit and backend correlation.

Risks:

- PostgreSQL native logs show axis_service.
- Database-level RBAC may be weakened.
- Compromise of AXIS service account is high impact.

Mitigations:

- Least-privilege service account.
- Per-environment credentials.
- Correlation ID injection via controlled session metadata.
- Strict block of client attempts to modify AXIS-managed metadata.
- Optional RLS/custom GUC strategy.

## Certificate Rotation

Enterprise mode must support:

- Reloading certs without dropping existing sessions where possible.
- New sessions using new certs.
- Expiry monitoring.
- mTLS CA bundle rotation.
- Operator-visible certificate health.

This is not optional for production. Certificates expire because apparently time itself enjoys breaking infrastructure.

## Current Known Weaknesses

- PoC pass-through auth is not identity verification.
- SCRAM strategy is unresolved.
- mTLS target requires operational tooling.
- Service account model requires strong attribution and monitoring.

## Acceptance Criteria

Auth/TLS strategy is acceptable only when AXIS can clearly say which identity it verified, which identity it merely observed, and how that distinction appears in audit evidence.
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---


# Policy and Audit Mapping

## Purpose

This document maps native PostgreSQL traffic into AXIS policy and audit concepts.

## AxisRequestEnvelope

Native PG mode must produce a normalized envelope for the existing policy engine.

Minimum fields:

```text
axis_request_id
session_id
connection_id
source_transport = pg_wire
client_addr
backend_addr
database
unauthenticated_claimed_db_user
verified_actor_id
service_name
tenant_id
application_name_client_claimed
application_name_axis_managed
query_mode
raw_sql_hash
sql_text_or_redacted_sql
normalized_sql
statement_kind
risk_classification
transaction_state
policy_id
policy_version
policy_sha256
received_at
```

## Identity Fields

Separate:

- `unauthenticated_claimed_db_user`: observed from client/startup/auth pass-through.
- `verified_actor_id`: identity AXIS verified through mTLS/OIDC/etc.
- `application_name_client_claimed`: client-provided value.
- `application_name_axis_managed`: AXIS-overwritten backend value.

Never collapse these fields. Compliance people love asking who verified what, because apparently evidence should mean things.

## Original Bytes and Metadata

User SQL must not be rewritten for correlation.

AXIS may issue separate backend session metadata commands, but those commands must be:

- Audit-recorded.
- Policy-classified as AXIS control statements.
- Protected from client overwrite.
- Not confused with user SQL.

## Audit Event Taxonomy

Required events:

```text
AXIS_CONNECTION_ACCEPTED
AXIS_BACKEND_CONNECTED
AXIS_STARTUP_PHASE_COMPLETED
AXIS_QUERY_RECEIVED
AXIS_POLICY_EVALUATED
AXIS_QUERY_FORWARDED
AXIS_BACKEND_COMMAND_COMPLETE
AXIS_BACKEND_RESPONSE_RECEIVED
AXIS_BLOCK_APPLIED
AXIS_APPROVAL_REQUIRED
AXIS_APPROVAL_TICKET_CREATED
AXIS_APPROVAL_TICKET_REUSED
AXIS_SAFETY_ROLLBACK_ISSUED
AXIS_TRANSACTION_STATE_CHANGED
AXIS_EXECUTION_UNKNOWN
AXIS_CLIENT_DELIVERY_UNKNOWN
AXIS_CONNECTION_POISONED
AXIS_CONNECTION_CLOSED
```

## Backend Did Not Receive Query

For BLOCK and APPROVAL_REQUIRED:

```json
{
  "backend_forwarded": false,
  "tcp_bytes_forwarded": 0,
  "forward_path_reachable": false,
  "decision_before_forward": true
}
```

This is an AXIS assertion. v1.2 tests must prove this with a backend mock or intercept layer, not only PostgreSQL query logs.

## CommandComplete Audit

When backend returns CommandComplete, AXIS should capture:

- command tag.
- row count if available.
- backend completion timestamp.
- transaction status after ReadyForQuery.
- backend response hash.

Example:

```json
{
  "event": "AXIS_BACKEND_COMMAND_COMPLETE",
  "command": "UPDATE",
  "row_count": 50000,
  "axis_request_id": "..."
}
```

## Execution Unknown vs Client Delivery Unknown

These are separate.

`EXECUTION_UNKNOWN`: AXIS forwarded to backend but cannot confirm backend completed.

`CLIENT_DELIVERY_UNKNOWN`: backend completed and AXIS observed completion, but client disconnected before receiving response.

These must not be merged. One is database uncertainty. The other is client delivery uncertainty.

## Parameter Handling

Raw bind parameters must not be written to audit by default. Use:

- redaction.
- hashing.
- allowlisted diagnostic capture.
- field-level sensitivity rules.

## Approval Linkage

Audit must answer:

- Who requested the operation?
- Which query caused approval?
- Who approved/rejected?
- Under which policy version?
- What exact execution was authorized?
- Did execution occur after approval or was a new evaluation required?

## Current Known Weaknesses

- Strong backend-side independent witnessing is not yet implemented.
- Parameter redaction rules need policy authoring support.
- CommandComplete row count extraction depends on protocol parsing.
- Approval idempotency is defined separately and must be implemented before enterprise pilot.

## Acceptance Criteria

An audit record is acceptable only if a reviewer can distinguish observed identity, verified identity, policy decision, backend reachability, execution confirmation, and delivery uncertainty.
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---


# PoC Scope

## Purpose

This document defines the first native PostgreSQL PoC and prevents scope creep.

## Goal

Prove that AXIS can sit between a PostgreSQL client and backend PostgreSQL, intercept Simple Query messages, evaluate policy before forwarding, block before backend reach, and produce audit evidence.

## Non-Goal

This PoC is not production-ready and not ORM-compatible.

## Supported

- psql or controlled test client.
- Cleartext TCP.
- Startup/auth pass-through.
- ParameterStatus pass-through.
- BackendKeyData pass-through only as PoC limitation.
- ReadyForQuery pass-through and generated ReadyForQuery for local idle BLOCK.
- Simple Query `Q`.
- ALLOW forwarding of original bytes.
- BLOCK before backend forward.
- APPROVAL_REQUIRED before backend forward.
- PostgreSQL-compatible ErrorResponse.
- Audit evidence.

## Blocked or Unsupported

- Extended Query.
- COPY.
- COPY FROM PROGRAM / COPY TO PROGRAM.
- TLS termination.
- SCRAM verification.
- CancelRequest.
- LISTEN/NOTIFY.
- Pipeline mode.
- Large objects.
- Replication protocol.
- PgBouncer transaction pooling.
- Connection pooling.
- Server spoofing beyond what is necessary.

## Connection Model

One client connection maps to one backend connection.

AXIS does not pool connections in PoC.

## Query Size Limit

AXIS must enforce a maximum SQL message size before unbounded allocation. The PoC must fail closed on over-limit messages.

## Transaction Behavior

PoC strict mode:

- BLOCK outside transaction: ErrorResponse + ReadyForQuery Idle + connection remains open.
- APPROVAL outside transaction: ErrorResponse + ticket + ReadyForQuery Idle + connection remains open.
- BLOCK inside transaction: ErrorResponse if possible + AXIS safety ROLLBACK + audit + close.
- APPROVAL inside transaction: ticket + safety ROLLBACK + audit + close.

## Acceptance Tests

PoC must prove:

- Safe SELECT ALLOW reaches backend.
- Dangerous DELETE BLOCK does not reach backend.
- APPROVAL_REQUIRED does not reach backend.
- Audit logs backend_forwarded=false for blocked/approval queries.
- Backend mock verifies zero forwarded bytes for blocked query payload.
- Policy failure fails closed.
- Audit WAL failure rejects protected write.
- Backend down returns controlled error.
- Oversized query fails closed before unbounded allocation.
- COPY is blocked.
- Extended Query receives FeatureNotSupported or controlled close.
- Transaction-internal BLOCK triggers safety rollback attempt and connection close.

## Current Known Weaknesses

- BackendKeyData pass-through makes CancelRequest unsafe outside lab.
- Auth pass-through does not verify identity.
- Simple Query excludes major ORMs.
- Performance results are not enterprise-representative.

## Exit Criteria

PoC is complete only when it proves enforcement semantics, not merely connectivity. “psql connects” is a cute demo, not a product milestone.
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---


# Failure Modes

## Purpose

This document defines how AXIS behaves under failure in Native PG mode.

## Rule

If AXIS is uncertain about safety, it must fail closed and preserve evidence where possible.

## Failure Matrix

| Failure | Behavior |
|---|---|
| Backend down before query | ErrorResponse, audit, no forward |
| Backend timeout before completion | EXECUTION_UNKNOWN if bytes sent |
| Audit WAL unavailable | Reject protected operations |
| Policy engine unavailable | BLOCK |
| Policy manifest invalid | Refuse readiness |
| Client disconnect before forward | audit disconnect, no backend action |
| Client disconnect after forward | backend may continue; audit state accordingly |
| Backend response lost | EXECUTION_UNKNOWN |
| Backend completed but client disconnected | CLIENT_DELIVERY_UNKNOWN |
| Partial write to backend | EXECUTION_UNKNOWN + close |
| Partial response from backend | close + audit |
| AXIS OOM/crash | covered by restart recovery semantics |
| Disk full | readiness fail + reject protected operations |
| Protocol unsupported | FeatureNotSupported or close |
| Transaction poisoned | safety rollback attempt + close |
| Backend rollback failure | TRANSACTION_RESET_FAILED + close |

## Recursive Audit Failure

If AXIS cannot write audit evidence, it cannot safely process protected writes.

Behavior:

1. Circuit breaker opens.
2. Readiness fails.
3. New protected operations are rejected.
4. Operator alert is emitted.
5. Recovery requires audit subsystem health restoration.

## Execution Unknown

Use when AXIS sent bytes to backend but cannot confirm whether backend completed.

Required fields:

```json
{
  "bytes_written_to_backend": 142,
  "backend_response_received": false,
  "cannot_confirm_non_execution": true
}
```

## Client Delivery Unknown

Use when backend completed but the client did not receive confirmation.

Required fields:

```json
{
  "backend_completed": true,
  "client_delivery_confirmed": false,
  "database_state_known": true
}
```

## Failure During Safety ROLLBACK

If AXIS attempts rollback and fails:

- Mark connection poisoned.
- Close client connection.
- Close backend connection if possible.
- Audit reset failure.
- Increment high-severity metric.
- Require operator visibility.

## Current Known Weaknesses

- AXIS crash cannot write events after death; recovery must reconcile incomplete evidence.
- Backend-side idle transaction cleanup must be configured.
- Some failures require manual verification.

## Acceptance Criteria

Every failure mode must produce either durable evidence or a documented absence-of-evidence state with operator guidance. Pretending the problem is solved because the process restarted is how humans keep inventing outages with nicer dashboards.
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
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# Protocol Fidelity Matrix

## Purpose

This document classifies PostgreSQL protocol and SQL features by AXIS Native PG support level.

No feature may be described as supported unless AXIS has explicit state handling, tests, failure behavior, audit mapping, and operator visibility for that feature.

## Support Classes

| Class | Meaning |
|---|---|
| Supported in PoC | Implemented and tested in lab Simple Query PoC |
| Blocked in PoC | Explicitly rejected fail-closed |
| Required Before Pilot | Must be implemented before real OLTP pilot |
| Future | Useful but not pilot-blocking |
| Unsupported | Should not be routed through AXIS |

## Protocol Matrix

| Feature | v1.2 Class | Behavior |
|---|---|---|
| StartupMessage | Supported in PoC | Pass-through to backend |
| Authentication pass-through | Supported in PoC | AXIS does not verify identity |
| PasswordMessage / SASL messages | Supported in PoC pass-through | Must not imply AXIS verified user |
| ParameterStatus | Supported in PoC pass-through | Future controlled override for protected metadata |
| BackendKeyData | PoC pass-through; pilot blocker | Production must not leak backend key directly |
| ReadyForQuery | Supported in PoC | Must track transaction byte |
| Simple Query `Q` | Supported in PoC | Intercept/evaluate/forward or block |
| Extended Query Parse | Required Before Pilot | Not supported in Simple Query PoC |
| Extended Query Bind | Required Before Pilot | Parameter handling required |
| Extended Query Execute | Required Before Pilot | Portal/statement state required |
| Sync | Required Before Pilot | Required for Extended Query lifecycle |
| Describe | Required Before Pilot | Needed for many drivers |
| Close | Required Before Pilot | Statement/portal cleanup |
| Flush | Required Before Pilot | Driver compatibility |
| COPY SQL command | Blocked in PoC | Block all COPY statements |
| COPY FROM PROGRAM | Blocked in PoC | Critical dangerous command |
| COPY TO PROGRAM | Blocked in PoC | Critical dangerous command |
| COPY subprotocol | Blocked in PoC | No COPY stream support |
| CancelRequest | Required Before Pilot | See 26-CANCELREQUEST-DESIGN |
| SSLRequest | Blocked/unsupported in PoC | TLS roadmap required |
| GSSENCRequest | Unsupported in PoC | Future enterprise review |
| NegotiateProtocolVersion | Pass-through if observed | Must be added to tests |
| NoticeResponse | Pass-through | Non-fatal backend notice |
| ErrorResponse from backend | Pass-through | Do not rewrite backend errors |
| ErrorResponse from AXIS | Supported | Must use defined field set |
| CommandComplete | Pass-through and audit row count | Important evidence source |
| RowDescription/DataRow | Pass-through | Do not synthesize result rows |
| LISTEN/NOTIFY | Unsupported in PoC | Async message complexity |
| Pipeline mode | Unsupported | Must reject or disable |
| Large objects | Block/future review | Often function-based bypass surface |
| Replication protocol | Unsupported | Not an AXIS target |

## SQL Feature Matrix

| SQL Feature | v1.2 Class | Behavior |
|---|---|---|
| SELECT simple safe | Supported | Policy evaluated |
| INSERT/UPDATE/DELETE | Policy evaluated | Protected writes require policy |
| DDL | Block or approval | DDL in transaction risky |
| Multi-statement Simple Query | Blocked in PoC | Entire message rejected |
| CTE with write | Block/evaluate as write | Parser must detect |
| SELECT INTO | Block/evaluate as write | Writes data |
| SELECT FOR UPDATE | Risky | Locking behavior |
| Function call unknown | Block unless allowlisted | Side effects/DoS risk |
| pg_sleep | Block/rate limit | DoS risk |
| set_config | Block for protected GUCs | Metadata tampering risk |
| SET ROLE | Block | Identity confusion |
| SET SESSION AUTHORIZATION | Block | Identity bypass |
| SET search_path | Block or strict allowlist | Function resolution risk |
| SET statement_timeout | Protected | May alter AXIS expectations |
| SET lock_timeout | Protected | May alter failure behavior |
| SET idle_in_transaction_session_timeout | Protected | State recovery dependency |
| DISCARD ALL | Block | Resets session state |
| RESET ALL | Block or guarded | Clears protected context |

## COPY Detection Requirement

COPY must be detected at SQL grammar/token level, not by naive substring matching.

Invalid methods:

- `starts_with("COPY")` only;
- regex over raw SQL;
- keyword search inside comments or strings;
- assuming COPY appears only as protocol submessages.

Required method:

- parse Simple Query into PostgreSQL-compatible statement structure;
- classify top-level COPY command;
- classify COPY PROGRAM variants;
- block before backend forward;
- audit `copy_blocked=true` and `copy_variant` if known.

Until parser confidence is adequate, any ambiguous COPY-like statement must fail closed.

## ErrorResponse Field Set

AXIS-generated ErrorResponse must include:

| Field | Required | Notes |
|---|---|---|
| `S` | Yes | localized severity: ERROR |
| `V` | Yes | non-localized severity: ERROR |
| `C` | Yes | exact SQLSTATE |
| `M` | Yes | stable message |
| `D` | Yes | includes `axis_request_id`, ticket if present |
| `H` | Yes | operator/app hint |
| `P` | Optional | only if precise position known |
| `s/t/c/d/n` | No by default | do not fake schema/table/column |
| `F/L/R` | No | do not expose internal source info |

## CommandComplete Audit

For ALLOW queries, AXIS must inspect backend `CommandComplete` tag where available and audit:

- command tag;
- rows affected if parseable;
- query fingerprint;
- policy decision;
- backend completed timestamp.

Example:

```json
{
  "event_type": "AXIS_BACKEND_COMPLETED",
  "axis_request_id": "req_...",
  "command_tag": "UPDATE 50000",
  "rows_affected": 50000,
  "backend_confirmed": true
}
```

Large row counts may trigger future post-execution alerting, but they do not retroactively block an already executed query. Reality has an annoying direction called time.

## Current Known Weaknesses

- Full PostgreSQL semantic equivalence is not guaranteed.
- Extended Query remains the largest compatibility gap.
- CancelRequest requires dedicated design.
- COPY detection depends on parser quality.
- Production TLS changes protocol behavior.

## Success Looks Like

Every protocol behavior has one of: supported, blocked, required, future, or unsupported. Nothing floats in the swamp of “probably works.”
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# Identity Attribution Strategy

## Purpose

This document defines how AXIS records, verifies, propagates, and protects actor identity in Native PG mode.

Identity attribution determines whether AXIS evidence can answer: who requested this operation, under what application context, under which database identity, and who approved it?

## Core Problem

In PoC auth pass-through mode, AXIS relays authentication traffic but does not verify the cryptographic authentication result itself.

Therefore the user field from StartupMessage is not a verified identity from AXIS's perspective.

It must be treated as:

```text
unauthenticated_claimed_db_user
```

not:

```text
authenticated_actor
```

This distinction must appear in audit records, UI labels, and reviewer-facing explanations. Otherwise a compliance reviewer will ask “who verified this identity?” and the answer will be an awkward silence, traditionally the most expensive answer.

## Identity Fields

| Field | Source | Trust Level |
|---|---|---|
| `unauthenticated_claimed_db_user` | StartupMessage user | Low in pass-through PoC |
| `client_addr` | TCP peer | Medium, network-dependent |
| `tls_client_subject` | mTLS cert | High if AXIS terminates mTLS |
| `axis_actor_id` | AXIS auth layer | High if AXIS verifies |
| `application_name_claimed` | Client parameter | Low |
| `application_name_axis` | AXIS-generated | High |
| `tenant_id` | AXIS-resolved or policy input | Depends on source |
| `approval_operator_id` | Control plane auth | High if operator auth verified |
| `axis_request_id` | AXIS-generated | High |

## Backend Service Account Risk

If AXIS connects to PostgreSQL using one service account, PostgreSQL native logs will show the service account, not the original user.

Known weakness:

- PostgreSQL logs and `pg_stat_activity` lose direct actor identity.
- Existing RLS policies based on `current_user` may break.
- pg_audit output may need AXIS correlation fields to be meaningful.

This is acceptable only if AXIS audit becomes the authoritative actor evidence layer and if PostgreSQL is configured to receive AXIS correlation metadata.

## RLS Compatibility

Many enterprise databases use Row Level Security based on database role or session variables.

Service account mode can break RLS if policies depend on `current_user`.

Mitigations:

1. Use custom GUCs such as `axis.actor_id`, `axis.tenant_id`, `axis.request_id`.
2. Rewrite RLS policies to read AXIS-provided GUCs through `current_setting('axis.actor_id', true)`.
3. Prevent clients from modifying AXIS-protected GUCs.
4. Add correlation to `application_name` for operational visibility.
5. Document that RLS integration requires customer-specific database policy review.

## Protected Metadata Injection

AXIS may need to send backend metadata such as:

```sql
SET application_name = 'axis req=req_... actor=...';
SELECT set_config('axis.request_id', 'req_...', false);
SELECT set_config('axis.actor_id', 'actor_...', false);
```

This creates tension with the Original Bytes Rule.

v1.2 decision:

- Original user query bytes must not be rewritten.
- AXIS may send separate AXIS-control metadata commands before or around user queries only if:
  - they are audit-recorded as AXIS control operations;
  - they are not confused with user SQL;
  - they cannot change user query semantics except for documented identity/correlation context;
  - protected GUCs cannot be overwritten by user SQL.

## Protected GUCs

The following must be protected from client modification:

- `axis.request_id`
- `axis.actor_id`
- `axis.tenant_id`
- `axis.policy_id`
- `axis.policy_version`
- `axis.approval_id`
- `application_name` when AXIS-managed mode is enabled
- `search_path` unless explicitly allowed
- `role` / `session_authorization`
- `statement_timeout`
- `lock_timeout`
- `idle_in_transaction_session_timeout`

Client attempts to alter protected GUCs must be blocked or require explicit policy allow.

## Application Name Rules

Client-provided `application_name` is useful but untrusted.

AXIS should record:

- `application_name_claimed` from client;
- `application_name_axis` generated by AXIS.

Backend should receive AXIS-controlled application name in production attribution mode.

Example:

```text
axis service=orders-api actor=user_123 request=req_abc policy=prod_main@2026-05-01.1
```

Do not let client-provided `application_name` overwrite AXIS correlation fields.

## Approval Attribution

Every approval must link:

- original request identity;
- request fingerprint;
- policy decision;
- approval ticket ID;
- approving operator ID;
- approval scope;
- approval expiry;
- retry execution request ID.

Approval evidence must answer:

- who requested;
- who approved;
- what exactly was approved;
- how long the approval was valid;
- whether the executed retry matched the approved scope.

## Identity Modes

| Mode | Description | Production Suitability |
|---|---|---|
| Pass-through PoC | AXIS relays auth, does not verify actor | Lab only |
| Service account + AXIS metadata | AXIS verifies actor externally, backend uses service account | Candidate |
| mTLS actor identity | Client cert maps to actor/service | Strong enterprise candidate |
| OIDC/LDAP broker | AXIS verifies identity via external IdP | Candidate with complexity |
| Per-actor backend connection | Backend current_user preserved | Strong but operationally heavy |

## Current Known Weaknesses

- Pass-through PoC audit identity is claimed, not verified by AXIS.
- Service account mode may break current_user-based RLS.
- Metadata propagation can conflict with strict original-byte purity.
- Customer database policies may need changes.

## Success Looks Like

A reviewer can distinguish claimed DB user, AXIS-verified actor, backend service account, and approving operator without squinting at a log file and whispering prayers.
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# Threat Model and Bypasses

## Purpose

This document defines security threats, bypass classes, and required mitigations for AXIS Native PG mode.

AXIS is a policy enforcement point. It is not a WAF and not a guarantee that ALLOW means a query is safe. ALLOW means only that the query matched configured AXIS policy under observed context.

## Core Assets

- Protected production database write path.
- Policy decision integrity.
- Audit evidence integrity.
- Approval integrity.
- Actor attribution.
- Backend non-execution proof for blocked operations.
- Session/transaction state consistency.

## Threat Classes

| Threat | Risk | Default Posture |
|---|---|---|
| Parser mismatch | AXIS and PostgreSQL interpret SQL differently | Fail closed on ambiguity |
| Multi-statement bypass | Hidden write behind safe statement | Block entire message in PoC |
| COPY bypass | SQL-level or protocol-level bulk data path | Block all COPY in PoC |
| COPY FROM/TO PROGRAM | Host command execution risk under privileged DB | Always high risk |
| Function side effects | SELECT invokes write/DoS/unsafe action | Block unknown functions |
| search_path manipulation | Function/table resolution changes | Block/protect |
| role switching | Identity confusion | Block |
| protected GUC overwrite | Audit/identity tampering | Block |
| Extended Query template/param abuse | State split across Parse/Bind/Execute | Production prerequisite |
| CancelRequest bypass | Out-of-band cancel path | Production prerequisite |
| Timing side-channel | Policy boundaries inferred from latency | Mitigate in pilot |
| Large query DoS | Memory exhaustion | Enforce pre-buffer limit |
| Policy latency spike | Queue buildup and crash | Backpressure/fail closed |
| Metrics leakage | Sensitive labels expose metadata | Cardinality guardrails |

## COPY Detection

COPY is dangerous in two forms:

1. SQL command inside Simple Query, e.g. `COPY table FROM ...`.
2. PostgreSQL COPY subprotocol after backend accepts COPY.

AXIS Simple Query PoC must block COPY before backend dispatch.

Detection requirements:

- top-level SQL command classification;
- COPY variant detection;
- PROGRAM variant detection;
- no substring matching inside comments or string literals;
- ambiguous parsing fails closed.

Examples requiring block:

```sql
COPY users FROM STDIN;
COPY users TO STDOUT;
COPY users FROM PROGRAM 'cat /etc/passwd';
COPY users TO PROGRAM 'curl attacker';
```

## Function Side Effects

A `SELECT` can be dangerous:

```sql
SELECT update_salary(1, 50000);
SELECT set_config('axis.actor_id','evil',false);
SELECT pg_sleep(100);
SELECT dblink_exec('...', 'DROP TABLE users');
```

v1.2 policy posture:

- unknown functions are risky by default;
- known safe functions may be allowlisted;
- known dangerous functions must be blocklisted;
- functions affecting GUCs, roles, file access, network access, locks, sleeps, advisory locks, or extensions are high risk;
- function volatility metadata may help but is not enough by itself.

Recommended policy fields:

```json
{
  "function_policy": {
    "default": "block_unknown",
    "allowlist": ["lower", "upper", "count", "now"],
    "blocklist": ["pg_sleep", "set_config", "dblink_exec", "lo_import", "lo_export"]
  }
}
```

## Dollar-Quoting and Encoding

PostgreSQL dollar-quoting can hide keywords inside strings:

```sql
SELECT $$ DROP TABLE users $$;
SELECT $tag$ COPY users FROM PROGRAM 'x' $tag$;
```

AXIS must not classify SQL by raw keyword search. Parser/corpus tests must cover:

- dollar-quoted strings;
- escaped strings;
- Unicode identifiers;
- comments;
- mixed casing;
- null byte rejection;
- client encoding behavior.

## Large Query DoS

AXIS must enforce limits before unbounded allocation.

Required limits:

- max message length;
- max SQL text length;
- max parameter count;
- max parameter bytes;
- max pipeline backlog once pipeline support exists;
- max connection count per client;
- max risky decisions per actor/window.

Reject oversized messages with a controlled ErrorResponse and audit `oversized_query_rejected=true`.

## Timing Side-Channel

If BLOCK returns much faster than ALLOW, attackers can infer protected resources.

Mitigations for pilot:

- minimum response floor for policy denials where feasible;
- jitter for repeated policy probing;
- rate limit suspicious blocked probes;
- avoid revealing matched sensitive table names in client ErrorResponse;
- keep detailed reason in operator audit, not client response.

Do not add giant random delays to production traffic as a substitute for security. That is not defense; that is turning latency into incense.

## ErrorResponse Data Leakage

Client-visible errors must not expose:

- raw policy internals;
- sensitive table names unless already obvious from request;
- raw parameter values;
- backend topology;
- policy version hash unless acceptable;
- internal source file/routine.

Detailed information belongs in audit/control plane, not untrusted client output.

## AXIS ALLOW Misuse

AXIS ALLOW is not a vulnerability scanner result. It does not mean:

- query is injection-free;
- query is business-safe;
- query has correct input validation;
- query should be trusted forever;
- query is harmless under all database settings.

This must appear in customer-facing integration documentation.

## Current Known Weaknesses

- Full semantic equivalence with PostgreSQL is hard.
- Function side effects require schema/catalog awareness.
- Timing side-channel mitigation must be balanced against latency.
- Schema awareness is required for strong table-level policy authoring.

## Success Looks Like

AXIS fails closed on ambiguous PostgreSQL behavior instead of confidently allowing a query it only half understood.

## Failure Looks Like

A regex sees `SELECT`, PostgreSQL executes a dangerous function, and everyone gathers around the incident report pretending this was unforeseeable.
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# Performance Budget

## Purpose

This document defines early performance targets for AXIS Native PG mode and clarifies that performance numbers are meaningless unless tied to durability mode, policy complexity, backend distance, and driver protocol.

## Core Warning

A single p50 overhead target is not a production claim.

AXIS performance depends on:

- network hop count;
- policy evaluation cost;
- SQL parsing cost;
- audit WAL append mode;
- fsync/group commit strategy;
- approval store latency;
- backend response time;
- connection count;
- workload mix;
- transaction duration;
- driver protocol behavior.

Pretending otherwise is a fine way to create sales collateral and a poor way to keep systems alive.

## Measurement Modes

| Mode | Meaning |
|---|---|
| Lab loopback | client, AXIS, backend on same host/network |
| Local network | same LAN/VPC |
| Kubernetes sidecar | localhost pod path |
| Gateway service | network hop through AXIS service |
| HA/multi-instance | load balancer and shared stores included |

Every benchmark must name its mode.

## Early Target Bands

These are engineering targets, not customer SLAs.

| Path | Lab Target | Early Realistic Target |
|---|---:|---:|
| ALLOW Simple Query p50 overhead | 2-8ms | 8-25ms |
| ALLOW Simple Query p95 overhead | 5-20ms | 30-70ms |
| BLOCK p50 overhead | 2-10ms | 8-30ms |
| APPROVAL_REQUIRED p50 overhead | 5-30ms | 20-100ms depending store |
| ErrorResponse generation | <2ms | <10ms |
| Policy eval local p95 | <5ms | <20ms |
| Audit append p95 | depends on durability mode | depends on durability mode |

Hard customer claims must not be made until measured against the customer topology.

## Required Metrics

- p50/p95/p99 total AXIS overhead.
- policy evaluation latency histogram.
- SQL classification latency histogram.
- audit append latency histogram.
- audit fsync/group commit latency.
- backend connect latency.
- backend response latency.
- client response latency.
- approval store latency.
- active connections.
- queue depth/backpressure state.
- dropped/rejected connections.
- execution_unknown count.

## Benchmark Scenarios

1. Direct PostgreSQL baseline.
2. AXIS ALLOW with audit durability strict.
3. AXIS ALLOW with group commit durability.
4. AXIS BLOCK with no backend forward.
5. AXIS APPROVAL_REQUIRED with ticket creation.
6. Backend timeout.
7. Audit WAL pressure.
8. High connection count.
9. Risky query flood.
10. Large query rejection.

## Audit Durability Dependency

Performance claims must name audit mode:

| Audit Mode | Latency | Evidence Risk |
|---|---|---|
| fsync per critical event | Highest | Strongest local durability |
| group commit | Lower | bounded crash window |
| async append without sync | Lowest | unacceptable for protected writes unless explicitly accepted |

For protected production writes, AXIS must not forward before at least a durable dispatch intent under the selected durability contract.

See `27-PERFORMANCE-AND-DURABILITY-TRADEOFFS.md`.

## Backpressure

If policy or audit latency spikes, AXIS must apply backpressure instead of unbounded queuing.

Required behavior:

- per-client connection limits;
- global active session limit;
- audit queue high-water mark;
- policy queue high-water mark;
- risky query rate limits;
- fail-closed rejection under overload;
- operator-visible overload state.

## Current Known Weaknesses

- v1.2 does not yet include measured benchmarks.
- Extended Query will add state and classification cost.
- Approval store HA may add latency.
- TLS/mTLS adds handshake and CPU cost.
- fsync strategy can dominate tail latency.

## Success Looks Like

AXIS publishes numbers with context and trade-offs: topology, workload, audit mode, policy complexity, and tail latency.

## Failure Looks Like

Someone writes “<2ms overhead” on a slide and later discovers fsync, TLS, policy evaluation, network hops, and reality. Reality, rude as ever, wins.
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---


# Application Integration Guide

## Purpose

This document explains what applications must expect when using AXIS Native PG mode.

## Honest Integration Claim

AXIS requires at minimum:

- connection string change.
- driver compatibility verification.
- protocol feature compatibility check.
- possible error-handling adaptation for approval workflows.
- TLS/certificate configuration in enterprise mode.

No claim of “zero application change” is allowed.

## Connection String

Example lab:

```text
postgres://user:pass@axis-host:6543/dbname?sslmode=disable
```

Production will differ.

## Approval Handling

APPROVAL_REQUIRED returns PostgreSQL ErrorResponse.

Applications should:

1. Detect AXIS approval condition through SQLSTATE and structured Detail.
2. Show or log approval ticket ID.
3. Avoid blind retry loops.
4. Retry only after approval and with idempotency guidance.
5. Discard connection if AXIS indicates connection poisoned.

## BLOCK Handling

BLOCK means policy denied execution before backend forward.

Applications should treat it as authorization failure, not database outage.

## Connection Poisoned

If AXIS returns a connection-poisoning error or closes the connection after transaction-internal BLOCK/APPROVAL, application pools must discard that connection.

Guide must include examples for:

- psycopg.
- node-postgres.
- JDBC/Hibernate.
- Npgsql.
- pgx.

## DDL and Migrations

DDL requiring approval should not be submitted inside large application transactions.

Migration tools such as Flyway, Liquibase, Alembic may wrap DDL in transactions. AXIS policy should either:

- require approval outside transaction.
- reject DDL-in-transaction with clear error.
- provide a migration workflow through HTTP/query or dedicated ops mode.

## ErrorResponse Fields

Applications should not parse ticket IDs out of free text if structured Detail is available.

AXIS should provide:

- SQLSTATE.
- Message.
- Detail with JSON-ish structured payload.
- Hint with human guidance.
- axis_request_id.

## Current Known Weaknesses

- ORM behavior varies.
- Simple Query PoC is not enough for most apps.
- Approval requires application awareness.
- Transaction reset behavior can trigger retries if app configuration is careless.

## Acceptance Criteria

The guide is acceptable only if a developer can understand what breaks, how to detect it, and when not to retry like a caffeinated raccoon.
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# Risk Register

## Purpose

This document tracks high-priority risks for AXIS Native PG mode. It must be updated at every major milestone.

## Severity Scale

| Severity | Meaning |
|---|---|
| Critical | Blocks production pilot or can cause unsafe execution/evidence failure |
| High | Major reliability/security concern requiring mitigation before broad pilot |
| Medium | Important but manageable with constraints |
| Low | Documented limitation or future improvement |

## Risk Register

| ID | Risk | Severity | Status | Mitigation |
|---|---|---:|---|---|
| R1 | Transaction divergence | Critical | Open | Strict rollback+close, future lenient design |
| R2 | Extended Query incompatibility | Critical | Open | EQ roadmap, driver matrix |
| R3 | CancelRequest unsupported | Critical | Open | Dedicated design in RFC 26 |
| R4 | Identity attribution loss | Critical | Open | Verified actor model, protected metadata |
| R5 | Audit WAL format unspecified | Critical | Addressed v1.2 | RFC 25 |
| R6 | Performance/durability contradiction | Critical | Addressed v1.2 | RFC 27 |
| R7 | Approval store single point of failure | Critical | Open | RFC 28 |
| R8 | Emergency bypass audit gap | Critical | Open | RFC 29 |
| R9 | Parser mismatch | Critical | Open | parser corpus, fail closed ambiguity |
| R10 | AXIS crash loses in-flight state | Critical | Open | restart recovery + durable dispatch intent |
| R11 | COPY FROM PROGRAM bypass | Critical | Open | grammar-level detection and block |
| R12 | Backend failover ambiguity | High | Open | primary health + manual/auto mode |
| R13 | PgBouncer transaction pooling mismatch | Critical | Open | unsupported; test rejection |
| R14 | Function side effects | High | Open | function allow/blocklist, schema awareness |
| R15 | Timing side-channel | Medium | Open | min latency/jitter/rate limit |
| R16 | ErrorResponse data leakage | High | Open | sanitized client errors |
| R17 | High-cardinality metrics overload | Medium | Open | observability guardrails |
| R18 | Audit WAL retention/disk growth | High | Open | retention/archive policy |
| R19 | Policy evaluation latency spike | High | Open | backpressure/fail-closed queue limits |
| R20 | Multi-AXIS clock skew | Medium | Open | monotonic store timestamps / expiry rules |
| R21 | ORM implicit prepared statements | Critical | Open | Extended Query support and compatibility matrix |
| R22 | Approval replay/dedup race | High | Open | RFC 18 + RFC 28 |
| R23 | RLS broken by service account | High | Open | protected GUC + customer DB review |
| R24 | Backend confirmed but client delivery unknown | High | Open | distinct audit event |
| R25 | Large query memory DoS | High | Open | pre-buffer message limits |
| R26 | Protected GUC tampering | High | Open | SET filtering |
| R27 | Cluster policy version drift | Critical | Open | policy version consistency gate |
| R28 | Shared approval store partition | Critical | Open | fail closed or single-writer guarantee |
| R29 | Audit-only bypass mode misunderstood as safe enforcement | Medium | Open | explicit UI/status mode |
| R30 | Sales overclaim | High | Open | customer-facing claims control |

## Required Review Cadence

- Before Simple Query PoC implementation.
- After Simple Query PoC passes tests.
- Before Extended Query implementation.
- Before external reviewer demo.
- Before customer pilot.
- After any incident or unexpected failure mode.

## Current Known Weaknesses

- Several Critical risks remain open.
- This risk register does not itself mitigate risk. Stunning, yes, but documents still do not execute code.

## Success Looks Like

No production pilot begins while Critical risks are marked open without explicit written exception.
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---


# Restart Recovery Semantics

## Purpose

This document defines how AXIS handles its own crash, restart, OOM kill, node eviction, or rolling upgrade.

## Problem

If AXIS dies mid-flight, it may be unable to write final audit events. Backend queries may continue, transactions may remain open, and clients may time out.

A graceful drain plan does not solve OOM kill.

## Required Session Markers

Before forwarding a protected operation, AXIS should write an in-flight marker:

```json
{
  "event": "AXIS_BACKEND_DISPATCH_INTENT",
  "axis_request_id": "...",
  "session_id": "...",
  "backend_addr": "...",
  "sql_hash": "...",
  "durably_written_before_forward": true
}
```

After completion:

```json
{
  "event": "AXIS_BACKEND_COMPLETION_OBSERVED",
  "axis_request_id": "..."
}
```

On startup, AXIS scans for dispatch intents without completion.

## Startup Recovery

At startup:

1. Validate audit WAL continuity.
2. Identify incomplete in-flight operations.
3. Mark them as `RECOVERY_INCOMPLETE_EXECUTION`.
4. Emit operator alert.
5. Refuse readiness if unresolved critical uncertainty exceeds policy threshold.
6. Rely on backend-side idle transaction timeout as mitigation for leaked sessions.

## Backend Connection Leaks

AXIS process death should close its TCP sockets, but backend may keep work alive briefly. PostgreSQL timeout settings must be part of deployment:

- statement_timeout.
- idle_in_transaction_session_timeout.
- lock_timeout.
- TCP keepalive settings.

## Rolling Upgrade

Graceful path:

1. Mark instance draining.
2. Stop accepting new connections.
3. Allow safe idle connections to close.
4. Force-close poisoned/long-running sessions after timeout.
5. Write drain audit event.
6. Upgrade.
7. Rejoin after readiness.

## Current Known Weaknesses

- AXIS cannot write after sudden death.
- Recovery may only identify uncertainty, not resolve it.
- Backend-side timeout configuration is required.
- Global recovery in multi-AXIS mode requires instance IDs.

## Acceptance Criteria

AXIS restart semantics are acceptable only if an operator can see which operations were in-flight and which require manual verification.
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---


# Approval Idempotency Model

## Purpose

This document defines how AXIS handles repeated APPROVAL_REQUIRED operations.

## Problem

Without idempotency, the same dangerous query can create many tickets, race approvals, and produce ambiguous authorization.

## Approval Scope

An approval must define exact scope:

```text
actor/service
tenant
database
environment
normalized_sql_hash
raw_sql_hash
policy_id
policy_version
risk_level
time_window
parameter_hashes_if_applicable
transaction_state
```

## Ticket Creation Rule

If an identical approval scope already has a pending ticket, AXIS should return the existing ticket unless policy requires one-ticket-per-attempt.

## Approval Reuse Rule

Approved tickets do not mean “always allow forever.”

Default:

- Approval authorizes one future matching execution attempt.
- Execution must re-evaluate policy.
- Policy version must match or ticket is invalid.
- Ticket expires.
- Ticket records consumed execution.

## Race Conditions

If two clients request the same dangerous operation:

- Same scope may deduplicate into one ticket.
- Different actor/service/tenant creates separate ticket.
- Approval consumption must be atomic.
- A consumed approval cannot be replayed.

## Rejection

Rejected tickets should not suppress future requests forever unless policy says so. Repeated rejected requests may trigger rate limiting.

## Audit Events

Required:

```text
AXIS_APPROVAL_TICKET_CREATED
AXIS_APPROVAL_TICKET_REUSED
AXIS_APPROVAL_TICKET_APPROVED
AXIS_APPROVAL_TICKET_REJECTED
AXIS_APPROVAL_TICKET_EXPIRED
AXIS_APPROVAL_TICKET_CONSUMED
AXIS_APPROVAL_SCOPE_MISMATCH
```

## Current Known Weaknesses

- Requires durable approval store.
- Requires concurrency-safe consumption.
- Extended Query parameters complicate scope.
- Approval UX must not encourage blind retries.

## Acceptance Criteria

A reviewer must be able to answer: what exactly was approved, by whom, for whom, under which policy, for how long, and whether it was used.
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# Emergency Bypass Procedure

## Purpose

This document defines how AXIS can be bypassed during emergencies without pretending the evidence chain remains intact.

Emergency bypass is a business continuity tool, not a security feature.

## Bypass Types

| Mode | Meaning | Evidence Impact |
|---|---|---|
| Graceful drain | Stop new connections, allow existing safe sessions to finish | Minimal gap |
| Hard bypass | Applications connect directly to PostgreSQL | AXIS blind gap |
| Audit-only bypass | AXIS remains inline but does not enforce selected policies | Evidence continues, enforcement reduced |
| Read-only emergency mode | Writes blocked, reads pass/audit | Safer degraded mode |
| Full outage | AXIS unavailable, manual direct DB access | Largest gap |

## Required Pre-BYPASS Event

If AXIS is running, it must write:

```json
{
  "event_type": "AXIS_EMERGENCY_BYPASS_DECLARED",
  "declared_by": "operator_id",
  "reason": "...",
  "mode": "hard_bypass|audit_only|read_only",
  "started_at": "...",
  "policy_version": "...",
  "audit_head_hash": "..."
}
```

If this cannot be written, the operator must record an external incident note. Yes, that is weaker. No, pretending otherwise does not strengthen evidence.

## Hard Bypass Procedure

1. Declare incident.
2. Export audit bundle if possible.
3. Record current audit head hash.
4. Freeze approval tickets if AXIS cannot enforce them.
5. Change connection string or routing to direct PostgreSQL.
6. Enable heightened PostgreSQL native logging.
7. Record bypass start timestamp.
8. Record operators involved.
9. Monitor direct database activity.
10. Restore AXIS when stable.
11. Reconcile bypass window.

## Audit-Only Mode

Audit-only mode is optional future behavior.

It means AXIS remains inline and records traffic but does not block according to selected policies.

Rules:

- UI/status must scream that enforcement is disabled/reduced.
- Audit events must include `enforcement_mode=audit_only`.
- Approval policies must be disabled or marked non-enforcing.
- Audit-only mode must require explicit operator authorization.
- Audit-only mode must have expiry.

Audit-only is not fail-open pretending to be safe. It is a declared degraded mode.

## Bypass Gap

When AXIS is hard-bypassed, AXIS cannot provide backend non-execution proof for the bypass window.

Required reconciliation inputs:

- PostgreSQL logs.
- pg_audit logs if available.
- connection pool logs.
- application logs.
- change ticket/incident record.
- AXIS pre-bypass and post-restore audit head hashes.

## Post-Bypass Reconciliation

Create an event:

```json
{
  "event_type": "AXIS_BYPASS_RECONCILIATION_COMPLETED",
  "bypass_window_start": "...",
  "bypass_window_end": "...",
  "postgres_log_sources": ["..."],
  "unverified_operations_count": 0,
  "manual_findings": "...",
  "operator_id": "..."
}
```

If operations cannot be verified, mark them as unverified. Do not launder uncertainty into confidence. That is how compliance paperwork becomes fiction.

## Emergency Bypass Acceptance Criteria

- Operator has a documented path to restore database access.
- AXIS evidence explicitly marks the gap.
- PostgreSQL logging is elevated during bypass.
- Approval semantics during bypass are defined.
- Customer understands that hard bypass breaks AXIS enforcement and AXIS evidence guarantees.

## Current Known Weaknesses

- Hard bypass creates unavoidable evidence gaps.
- Audit-only mode is not implemented in Simple Query PoC.
- Reconciliation depends on customer PostgreSQL logging quality.

## Success Looks Like

During an incident, operators know exactly how to bypass AXIS and exactly what evidence guarantees are lost.

## Failure Looks Like

AXIS is bypassed at 02:13, nobody records it, and at 09:00 everyone pretends the audit chain is continuous. Charming, in the way building collapses are charming.
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# Operability and Observability

## Purpose

This document defines metrics, logs, traces, dashboards, health checks, diagnostics, and on-call workflows required for AXIS Native PG mode.

AXIS is not operable if an on-call engineer cannot answer within minutes:

- why was this query blocked?
- did it reach backend?
- which policy version decided?
- is audit healthy?
- is the connection state safe?
- are we in enforcement, audit-only, bypass, or degraded mode?

## Required Metrics

### Connection Metrics

- `axis_active_client_connections`
- `axis_active_backend_connections`
- `axis_connection_accept_rate`
- `axis_connection_reject_rate`
- `axis_connection_close_reason_total`
- `axis_backend_connect_errors_total`
- `axis_connection_poisoned_total`

### Decision Metrics

- `axis_policy_allow_total`
- `axis_policy_block_total`
- `axis_policy_approval_required_total`
- `axis_policy_eval_latency_seconds_bucket`
- `axis_policy_eval_queue_depth`
- `axis_policy_timeout_total`

### Audit Metrics

- `axis_audit_append_latency_seconds_bucket`
- `axis_audit_fsync_latency_seconds_bucket`
- `axis_audit_queue_depth`
- `axis_audit_circuit_breaker_state`
- `axis_audit_wal_bytes_total`
- `axis_audit_wal_disk_free_bytes`
- `axis_audit_export_total`
- `axis_audit_verify_failure_total`

### PostgreSQL Protocol Metrics

- `axis_simple_query_total`
- `axis_extended_query_rejected_total`
- `axis_copy_blocked_total`
- `axis_cancel_request_total`
- `axis_unsupported_protocol_total`
- `axis_ready_for_query_state_total{state="I|T|E"}`
- `axis_command_complete_total`

### Failure Metrics

- `axis_execution_unknown_total`
- `axis_backend_confirmed_client_delivery_unknown_total`
- `axis_safety_rollback_attempt_total`
- `axis_safety_rollback_failed_total`
- `axis_restart_recovery_incomplete_total`
- `axis_bypass_declared_total`

## High-Cardinality Guardrails

Do not use the following as Prometheus labels:

- `axis_request_id`
- raw SQL
- SQL fingerprint if high-volume/unbounded
- raw actor ID
- ticket ID
- client IP if high-cardinality in multi-tenant deployments
- table names unless bounded and explicitly accepted

Use logs/traces for request-scoped values. Metrics must survive production cardinality; otherwise Prometheus becomes a bonfire with an HTTP port.

## Required Structured Logs

Every decision log should include:

- timestamp;
- level;
- event_type;
- axis_request_id;
- session_id;
- policy_id;
- policy_version;
- decision;
- risk;
- backend_forwarded;
- tcp_bytes_forwarded;
- transaction_state;
- actor context where safe;
- error class;
- evidence hash pointer.

Raw bind values and raw sensitive SQL must not be logged by default.

## Required Traces

OpenTelemetry spans:

```text
axis.session
  axis.startup
  axis.query.receive
  axis.classify
  axis.policy.evaluate
  axis.audit.intent_append
  axis.backend.forward
  axis.backend.response
  axis.audit.completion_append
  axis.client.respond
```

For rollback:

```text
axis.safety_rollback.issue
axis.safety_rollback.result
```

For approval:

```text
axis.approval.lookup_or_create
axis.approval.response
```

Span links should connect approval ticket creation to retry execution.

## Required Dashboards

### Operator Overview

- enforcement mode;
- audit health;
- backend health;
- policy version;
- active connections;
- decision rates;
- p95/p99 policy and audit latency;
- current failure count.

### Query Forensics

Given `axis_request_id`, show:

- received timestamp;
- actor context;
- SQL fingerprint;
- decision;
- matched rule;
- policy version;
- backend reached yes/no;
- bytes forwarded;
- backend command complete if any;
- audit hash chain pointer;
- approval ticket if any.

### Transaction State

- active sessions by state;
- InTransaction count;
- FailedTransaction count;
- PoisonedByAXIS count;
- safety rollback attempts;
- long-lived transaction list.

### Execution Unknown

- latest execution unknown events;
- backend dispatch intent status;
- manual reconciliation status;
- affected actor/application;
- suggested operator action.

## Health Checks

Readiness fails if:

- audit WAL cannot be appended;
- audit circuit breaker open;
- policy manifest invalid;
- backend unavailable in strict backend-required mode;
- approval store unavailable and approval policies active;
- instance policy version stale;
- disk below audit low-water mark;
- cert expired/invalid in TLS mode;
- node is draining;
- mode is hard bypass.

Liveness should not restart AXIS simply because backend is down. Restarting a healthy process because its dependency is down is how orchestration systems cosplay as chaos monkeys.

## Diagnostic Commands / Views

Future CLI/API should provide:

- `axisctl sessions list`
- `axisctl sessions show <session_id>`
- `axisctl request show <axis_request_id>`
- `axisctl audit head`
- `axisctl audit verify --since <time>`
- `axisctl approvals show <ticket>`
- `axisctl mode status`
- `axisctl drain start`
- `axisctl drain status`

## Current Known Weaknesses

- Dashboards are not part of lab Simple Query PoC unless scoped.
- On-call runbooks must be written before customer pilot.
- Metrics can leak information if label design is careless.
- OpenTelemetry integration adds overhead and must be sampled.

## Success Looks Like

An on-call engineer can diagnose a blocked or unknown query in under three minutes without SSH-ing into random containers and sacrificing dignity to grep.

## Failure Looks Like

The only observable signal is “app got 500.” That is not observability. That is a haunted house.
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---


# RFC Lifecycle and Versioning

## Purpose

This document defines how the native PG RFC set evolves.

## Problem

RFCs rot. Implementation changes. Drivers change. PostgreSQL versions change. Six months later, everyone quotes a stale document in a meeting and calls it architecture. Charming, but no.

## Versioning

RFC versions:

- v1.0: initial design.
- v1.2: production-realism hardening.
- v1.2: post-PoC update.
- v2.0: Extended Query design accepted.
- v3.0: production pilot readiness.

## Required Metadata

Each document must include:

- status.
- applies_to.
- last_reviewed.
- owner.
- implementation_status.
- supersedes.
- related_tests.

## Change Control

RFC changes required when:

- protocol support changes.
- policy behavior changes.
- transaction behavior changes.
- audit event taxonomy changes.
- deployment topology changes.
- approval semantics change.
- performance claims change.
- customer pilot exposes mismatch.

## Review Gates

- Before PoC implementation.
- After PoC completion.
- Before Extended Query implementation.
- Before external reviewer package.
- Before customer pilot.
- After first incident or near miss.

## Current Known Weaknesses

- This v1.2 set still has draft status.
- Implementation may reveal missing protocol details.
- Responsibility owner must be assigned before team growth.

## Acceptance Criteria

No implementation merge may contradict RFC without updating RFC or explicitly recording a deviation.
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---


# Extended Query Roadmap

## Purpose

This document defines the path from Simple Query PoC to modern ORM compatibility.

## Why Extended Query Is Not Optional

Most serious production stacks use prepared statements and Extended Query flows. Simple Query PoC is transport validation, not production readiness.

## Phases

### Phase EQ-0: Discovery

- Test major drivers.
- Record protocol defaults.
- Identify simple-query downgrade options.
- Document which stacks cannot downgrade.

### Phase EQ-1: Parse Tracking

Support Parse messages:

- capture statement name.
- capture SQL template.
- classify template.
- store statement state per session.
- block dangerous templates.

### Phase EQ-2: Bind Tracking

Support Bind messages:

- capture portal.
- associate with statement.
- hash/redact parameter values.
- enforce parameter count/format.
- prevent raw sensitive audit leakage.

### Phase EQ-3: Execute

Support Execute:

- evaluate combined statement context.
- enforce policy before backend Execute.
- audit parameter hashes.
- handle portals and result streaming.

### Phase EQ-4: Sync and Error Recovery

- correct transaction status.
- correct ReadyForQuery handling.
- recover from failed statement state.
- Close statement/portal support.

### Phase EQ-5: Driver Compatibility

Test:

- psycopg3.
- asyncpg.
- node-postgres.
- Prisma.
- JDBC/Hibernate.
- Npgsql.
- Go pgx.
- SQLAlchemy.

## Policy Model

Policy may evaluate:

- SQL template.
- statement kind.
- parameter count.
- parameter type.
- parameter hashes.
- actor/tenant.
- transaction state.

Raw parameter values are not stored by default.

## Current Known Weaknesses

- Full semantic evaluation with parameters is complex.
- Prepared statement state can outlive assumptions.
- Portals and cursors introduce statefulness.
- Driver quirks will be annoying because software enjoys humiliation.

## Acceptance Criteria

Extended Query is accepted only when at least four major driver stacks pass ALLOW/BLOCK/APPROVAL/audit tests without unsafe downgrades.
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# Cluster, Failover, and Multi-AXIS Semantics

## Purpose

This document defines how AXIS behaves when multiple AXIS instances, backend failover, shared approval stores, and policy consistency enter the design.

Single-node assumptions must not leak into enterprise pilot architecture.

## Deployment Modes

| Mode | Status |
|---|---|
| Single AXIS lab | Supported for PoC |
| Single AXIS pilot | Acceptable only with explicit customer risk acceptance |
| Multi-AXIS active/passive | Candidate |
| Multi-AXIS active/active | Requires shared stores and consistency checks |
| Kubernetes sidecar per app | Candidate with local scope |
| Central gateway | Requires HA and drain model |

## Policy Version Consistency

Every AXIS instance must expose:

- active policy id;
- active policy version;
- policy sha256;
- manifest sha256;
- activation time;
- instance id.

Load balancers or readiness probes should remove instances whose policy version is stale.

Requests must audit policy version per decision. In multi-AXIS mode, a request cannot rely on “cluster policy” as a vague concept. Vague concepts do not survive subpoenas.

## Multi-AXIS Approval Consistency

Approval tickets must be globally consistent across AXIS instances.

Requirements:

- shared durable store or single-writer approval service;
- unique ticket ID generation;
- idempotency key uniqueness;
- approval scope hash;
- expiry based on store time, not instance wall clock;
- compare-and-set for resolution;
- immutable resolved state;
- audit correlation from all instances.

If approval store is partitioned, approval-required operations must fail closed or route to single available writer. Duplicate approvals for the same scope must not silently authorize duplicate executions.

See `28-APPROVAL-STORE-HA-AND-CONSISTENCY.md`.

## Clock Skew

Do not use individual AXIS node clocks as sole authority for approval expiry or policy activation in active/active mode.

Mitigations:

- use store-side timestamps;
- include monotonic version counters;
- tolerate bounded skew;
- record node timestamp and store timestamp separately.

## Backend Failover

v1.2 defines two modes.

### Manual Failover Mode

AXIS points to a configured primary endpoint. If backend becomes unavailable:

- new writes fail closed;
- reads may fail or route according to policy;
- operator updates backend endpoint;
- AXIS drains/reconnects;
- audit records backend failover event.

Manual mode is simpler and safer for early pilots.

### Automatic Failover Mode

AXIS uses a resolver/service discovery mechanism to identify the current primary.

Requirements:

- primary health check;
- write target verification;
- read-only detection;
- stale primary prevention;
- split-brain handling;
- audit failover event;
- operator alert.

Automatic failover must not route writes to a replica because “it was reachable.” Reachability is not authority. This remains difficult for humans, load balancers, and apparently civilization.

## Read Replica Behavior

If AXIS supports read routing in future:

- policy must classify read vs write safely;
- transaction state must remain consistent;
- reads after writes may require primary;
- replica lag must be visible;
- SELECT functions with side effects must not be routed as harmless reads.

Read routing is not part of Simple Query PoC.

## PgBouncer Compatibility

| PgBouncer Mode | v1.2 Status |
|---|---|
| No PgBouncer | Lab supported |
| PgBouncer behind AXIS, session pooling | Candidate with tests |
| PgBouncer behind AXIS, transaction pooling | Unsupported |
| PgBouncer before AXIS | Not recommended |

### Session Pooling Reset Risk

PgBouncer session pooling can run `server_reset_query` such as `DISCARD ALL` or `RESET ALL` when returning backend connections.

AXIS must know whether reset commands occur and how they affect:

- prepared statements;
- protected GUCs;
- application_name;
- axis_request_id correlation;
- transaction state;
- backend session identity.

This must be tested before PgBouncer session pooling is accepted.

## Audit Chain in Multi-AXIS

Options:

| Option | Pros | Cons |
|---|---|---|
| Per-instance audit chain | Simple, local integrity | Requires bundle correlation |
| Central audit writer | Easier global order | New bottleneck/SPOF |
| Per-instance + external witness | Scalable and verifiable | More complex |

v1.2 recommendation:

- keep per-instance WAL chains;
- periodically export Merkle roots/external witness entries;
- correlate via cluster event index.

## Current Known Weaknesses

- Automatic backend failover is not implemented.
- Active/active approval consistency requires a real shared store design.
- Multi-instance global audit ordering is not solved by local hash chains.
- PgBouncer transaction pooling is unsupported.

## Success Looks Like

A query decision remains explainable even when traffic crosses multiple AXIS instances, backend primary changes, and approval state is shared.

## Failure Looks Like

Two AXIS nodes approve the same dangerous operation differently because one had yesterday’s policy and the other had today’s optimism.
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---


# Policy Authoring for Native PG Mode

## Purpose

This document defines policy authoring guidance for native PostgreSQL traffic.

## Why Native PG Policy Is Different

HTTP/query mode receives explicit request context. Native PG mode must infer context from protocol, session, identity, database, and SQL behavior.

Policy authors must not assume every SELECT is safe or every user string is verified.

## Required Policy Dimensions

Policies should support:

- environment.
- database.
- schema.
- table.
- operation kind.
- statement risk.
- actor/service.
- tenant.
- transaction state.
- query mode.
- policy version.
- function/procedure calls.
- COPY operations.
- SET/role/search_path commands.
- multi-statement behavior.
- approval requirements.

## Default Rules

Recommended defaults:

- Allow simple reads only if no write-like behavior.
- Block multi-statement by default.
- Block COPY by default.
- Block COPY FROM PROGRAM always.
- Block unsupported protocol states.
- Block DDL unless explicitly approved.
- Block SET ROLE unless explicitly allowed.
- Block protected GUC overwrite.
- Treat unknown functions as risky in production.
- Require approval for broad UPDATE/DELETE without safe predicate.
- Fail closed on parser uncertainty.

## Approval Policies

Approval policies must define:

- who may approve.
- approval scope.
- expiration.
- one-time vs reusable.
- whether transaction context invalidates approval.
- whether policy version changes invalidate approval.

## Testing Policies

Every native PG policy must have regression tests:

- ALLOW cases.
- BLOCK cases.
- APPROVAL cases.
- parser bypass corpus.
- transaction context cases.
- function/procedure cases.
- COPY cases.
- SET/search_path cases.

## Current Known Weaknesses

- Policy engine may need schema awareness.
- Function side effects are hard to classify.
- Extended Query parameters complicate authoring.
- RLS integration is future work.

## Acceptance Criteria

A native PG policy is acceptable only if it states what it allows, what it blocks, what it sends to approval, and what it does when it cannot classify.
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# Audit WAL Format and Evidence Spec

## Purpose

This document specifies the AXIS audit WAL entry format, hash-chain semantics, canonicalization rules, export format, and verification procedure for Native PG mode.

If AXIS claims durable evidence, the evidence format must be explicit. “We logged it somewhere” is not evidence; it is a diary with infrastructure ambitions.

## Scope

This spec covers:

- WAL line format;
- required event fields;
- hash algorithm;
- canonical JSON rules;
- chain continuity;
- dispatch intent and completion events;
- backend non-forward proof fields;
- export bundle format;
- verification procedure;
- retention and external witnessing.

## WAL Record Format

Each WAL record is one canonical JSON object encoded as UTF-8 and terminated by `\n`.

No pretty printing. No comments. No trailing commas. Apparently this must be said.

Minimum envelope:

```json
{
  "schema_version": "axis.audit.wal.v1",
  "event_id": "evt_...",
  "event_type": "AXIS_POLICY_EVALUATED",
  "event_time_utc": "2026-05-21T15:30:00.000000Z",
  "instance_id": "axis-node-1",
  "session_id": "sess_...",
  "axis_request_id": "req_...",
  "sequence": 12345,
  "previous_event_hash": "hex...",
  "event_body": {},
  "event_hash": "hex..."
}
```

## Canonicalization

Hash input is canonical JSON with:

- UTF-8 encoding;
- sorted object keys;
- no insignificant whitespace;
- stable number representation;
- `event_hash` excluded from its own hash input;
- `previous_event_hash` included;
- arrays in original semantic order.

Canonical hash input:

```text
canonical_json(record_without_event_hash)
```

Hash algorithm:

```text
SHA-256(canonical_json(record_without_event_hash))
```

Encoding:

```text
lowercase hex
```

## Required Common Fields

| Field | Required | Meaning |
|---|---|---|
| `schema_version` | yes | WAL schema identifier |
| `event_id` | yes | globally unique event ID |
| `event_type` | yes | event taxonomy |
| `event_time_utc` | yes | timestamp from AXIS node |
| `instance_id` | yes | AXIS instance |
| `session_id` | conditional | connection/session ID |
| `axis_request_id` | conditional | request correlation |
| `sequence` | yes | local monotonic sequence |
| `previous_event_hash` | yes | prior WAL event hash or genesis marker |
| `event_body` | yes | event-specific payload |
| `event_hash` | yes | SHA-256 hash |

## Event Taxonomy

Required Native PG event types:

- `AXIS_CONNECTION_ACCEPTED`
- `AXIS_STARTUP_PASSTHROUGH`
- `AXIS_AUTH_PASSTHROUGH_OBSERVED`
- `AXIS_READY_FOR_QUERY_OBSERVED`
- `AXIS_QUERY_RECEIVED`
- `AXIS_POLICY_EVALUATED`
- `AXIS_BACKEND_DISPATCH_INTENT`
- `AXIS_BACKEND_DISPATCHED`
- `AXIS_BACKEND_COMPLETED`
- `AXIS_BACKEND_FAILED`
- `AXIS_BLOCK_APPLIED`
- `AXIS_APPROVAL_ISSUED`
- `AXIS_SAFETY_ROLLBACK_ISSUED`
- `AXIS_SAFETY_ROLLBACK_RESULT`
- `AXIS_EXECUTION_UNKNOWN`
- `AXIS_BACKEND_CONFIRMED_CLIENT_DELIVERY_UNKNOWN`
- `AXIS_CONNECTION_CLOSED`
- `AXIS_EMERGENCY_BYPASS_DECLARED`
- `AXIS_BYPASS_RECONCILIATION_COMPLETED`
- `AXIS_RESTART_RECOVERY_INCOMPLETE`

## Query Received Body

```json
{
  "sql_hash": "sha256:...",
  "sql_fingerprint": "...",
  "raw_sql_redacted": null,
  "query_mode": "simple",
  "message_len_bytes": 128,
  "transaction_state_before": "Idle",
  "unauthenticated_claimed_db_user": "app_user",
  "axis_actor_id": null,
  "client_addr_hash": "...",
  "target_database": "prod"
}
```

Raw SQL must not be stored by default unless explicitly enabled and redaction policy accepts it.

## Policy Evaluated Body

```json
{
  "decision": "BLOCK",
  "reason": "dangerous_delete_without_where",
  "risk": "CRITICAL",
  "matched_rule": "block-dangerous-delete",
  "policy_id": "prod-main",
  "policy_version": "prod_main@2026-05-01.1",
  "policy_sha256": "...",
  "approval_required": false,
  "backend_forward_allowed": false
}
```

## Dispatch Intent Body

Before forwarding protected execution, AXIS writes intent:

```json
{
  "dispatch_id": "dispatch_...",
  "axis_request_id": "req_...",
  "backend_addr": "hash-or-label",
  "bytes_planned": 128,
  "durability_mode": "fsync_per_critical_event|group_commit",
  "intent_durable": true
}
```

If AXIS crashes after intent but before completion, restart recovery can mark incomplete execution.

## Block Applied Body

```json
{
  "decision": "BLOCK",
  "backend_forwarded": false,
  "tcp_bytes_forwarded": 0,
  "proof_scope": "AXIS internal forwarder assertion",
  "independent_backend_witness": false,
  "error_sqlstate": "42501"
}
```

Important: `tcp_bytes_forwarded=0` is an AXIS-layer assertion. It is strengthened by byte-level tests and code structure, not magically witnessed by PostgreSQL.

## Backend Completed Body

```json
{
  "dispatch_id": "dispatch_...",
  "backend_confirmed": true,
  "command_tag": "UPDATE 12",
  "rows_affected": 12,
  "ready_for_query_status": "I",
  "client_delivery_confirmed": true
}
```

If backend completed but client delivery failed:

```json
{
  "event_type": "AXIS_BACKEND_CONFIRMED_CLIENT_DELIVERY_UNKNOWN",
  "event_body": {
    "backend_confirmed": true,
    "client_delivery_confirmed": false,
    "command_tag": "UPDATE 12",
    "operator_action": "application_may_retry_duplicate_operation"
  }
}
```

This is distinct from `AXIS_EXECUTION_UNKNOWN`.

## Execution Unknown Body

```json
{
  "dispatch_id": "dispatch_...",
  "bytes_written_to_backend": 128,
  "backend_response_received": false,
  "tcp_error": "connection_reset",
  "cannot_confirm_non_execution": true,
  "operator_action": "manual_backend_reconciliation_required"
}
```

## Export Bundle Format

An audit export bundle must include:

```text
bundle.json
wal-segment-000001.jsonl
wal-segment-000002.jsonl
MANIFEST_SHA256.json
SIGNATURE.json optional
```

`bundle.json`:

```json
{
  "bundle_schema": "axis.audit.bundle.v1",
  "export_id": "export_...",
  "created_at_utc": "...",
  "instance_id": "axis-node-1",
  "range_start_sequence": 1,
  "range_end_sequence": 1000,
  "range_start_hash": "...",
  "range_end_hash": "...",
  "policy_versions_seen": ["..."],
  "signature_algorithm": "none|ed25519",
  "external_witness": null
}
```

## Verification Procedure

Verifier must:

1. Parse every JSONL record.
2. Validate schema version.
3. Recompute each event hash.
4. Verify `previous_event_hash` continuity.
5. Verify sequence monotonicity.
6. Verify manifest file hashes.
7. Verify signature if present.
8. Report gaps, duplicates, invalid hashes, malformed records.

Verification output must be machine-readable.

## External Witnessing

Future production hardening should periodically publish Merkle roots or head hashes to an external witness:

- customer-controlled object store;
- timestamping service;
- signed transparency log;
- append-only external ledger.

This reduces the “you wrote logs after the fact” objection.

## Retention and Pruning

Audit WAL can grow quickly.

Required future retention policy:

- hot WAL retention window;
- archived bundle schedule;
- compression;
- integrity verification before deletion;
- legal hold support;
- disk low-watermark alert;
- retention per customer contract.

Never prune unexported WAL silently.

## Current Known Weaknesses

- External witness is future work.
- Signature scheme may be disabled in early PoC.
- Raw SQL storage is disabled by default, which may limit forensic readability.
- Per-instance chains require bundle-level correlation in multi-AXIS deployments.

## Success Looks Like

An auditor can verify that event N follows event N-1, that a blocked query was never dispatched by AXIS, and that the bundle was not quietly edited after export.

## Failure Looks Like

A compliance reviewer asks for the evidence format and receives “it’s JSON somewhere.” The room temperature drops five degrees.
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# CancelRequest Design

## Purpose

This document defines how AXIS must handle PostgreSQL CancelRequest before production pilot.

CancelRequest is not a normal query message. It arrives on a separate connection and uses BackendKeyData `(pid, secret_key)` issued during startup. Mishandling it causes either broken cancellation or a security bypass.

## Why It Matters

Without CancelRequest support:

- applications cannot cancel long-running queries correctly;
- real backend cancellation keys may leak to clients;
- clients may bypass AXIS to cancel backend sessions;
- attackers may use leaked keys to disrupt other sessions;
- operators lose control during incidents.

## PoC Status

Simple Query PoC may reject or log CancelRequest as unsupported.

Production pilot must not begin until one of these is true:

1. CancelRequest mapping is implemented and tested; or
2. customer explicitly accepts that cancel is unsupported and config prevents backend key leakage.

Option 2 is unlikely to survive a serious reviewer, but optimism has injured systems before.

## Required Design

AXIS must not pass real backend `BackendKeyData` directly to the client in production mode.

Instead:

1. Backend sends real `(backend_pid, backend_secret)` to AXIS.
2. AXIS stores mapping internally.
3. AXIS generates fake `(axis_pid, axis_secret)`.
4. AXIS sends fake BackendKeyData to client.
5. Client later opens CancelRequest with fake key.
6. AXIS validates fake key.
7. AXIS looks up real backend key.
8. AXIS opens backend CancelRequest connection using real key.
9. AXIS audits cancel request and result.

## Mapping Table

```text
axis_pid, axis_secret
  -> session_id
  -> backend_addr
  -> backend_pid, backend_secret
  -> created_at
  -> expires_at
  -> state
```

State values:

- active;
- cancel_in_progress;
- cancelled;
- expired;
- backend_closed;
- unknown_after_restart.

## Security Requirements

- Fake secrets must be cryptographically random.
- Mapping must be per session.
- Mapping must expire when session closes.
- Mapping must not be reused.
- CancelRequest must be rate-limited.
- Failed cancel attempts must be audited.
- Real backend keys must never appear in client logs or ErrorResponse.

## Restart Behavior

Cancel mappings are process memory state unless persisted.

v1.2 recommendation:

- mappings are memory-only for early implementation;
- AXIS restart invalidates fake keys;
- clients attempting cancel after AXIS restart receive failure;
- backend-side timeout settings mitigate abandoned long queries;
- future production may persist encrypted mappings with strict expiry.

## Audit Events

Required events:

- `AXIS_BACKEND_KEYDATA_INTERCEPTED`
- `AXIS_BACKEND_KEYDATA_SUBSTITUTED`
- `AXIS_CANCEL_REQUEST_RECEIVED`
- `AXIS_CANCEL_REQUEST_VALIDATED`
- `AXIS_CANCEL_FORWARDED_TO_BACKEND`
- `AXIS_CANCEL_BACKEND_RESULT`
- `AXIS_CANCEL_REJECTED`

Example:

```json
{
  "event_type": "AXIS_CANCEL_REQUEST_RECEIVED",
  "event_body": {
    "axis_pid": 12345,
    "session_id": "sess_...",
    "validated": true,
    "forwarded_to_backend": true
  }
}
```

## Failure Modes

| Failure | Behavior |
|---|---|
| Unknown fake key | Reject, audit |
| Expired key | Reject, audit |
| Backend session closed | Reject or no-op, audit |
| Backend cancel connection fails | Audit cancel_unknown |
| AXIS restart lost mapping | Reject and suggest backend timeout/operator action |
| Cancel flood | Rate limit |

## Tests

- client receives fake BackendKeyData;
- real backend key never reaches client;
- valid fake cancel forwards real cancel;
- invalid fake cancel rejected;
- expired mapping rejected;
- closed session mapping removed;
- restart invalidates mapping;
- concurrent cancels are idempotent;
- audit sequence is complete.

## Current Known Weaknesses

- Memory-only mappings break cancel after AXIS restart.
- Persisted mappings require secret protection.
- Cancel outcome may still be timing-dependent.

## Success Looks Like

Clients can cancel long-running queries through AXIS without learning backend cancellation secrets.

## Failure Looks Like

AXIS passes through BackendKeyData, then calls itself a security control while handing out backend cancel keys like party favors.
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# Performance and Durability Trade-offs

## Purpose

This document defines the relationship between AXIS latency, audit durability, crash windows, and acceptable evidence guarantees.

Performance cannot be discussed honestly without durability mode. Any benchmark that omits audit fsync behavior is mostly decorative.

## Durability Modes

| Mode | Protected Write Forwarding Rule | Latency | Crash Window | Use |
|---|---|---:|---:|---|
| Strict Fsync | intent fsynced before forward | Highest | Minimal | High-assurance deployments |
| Group Commit | intent appended and fsynced within configured window | Medium | bounded | Early production candidate with customer acceptance |
| Async Audit | intent buffered only | Lowest | unbounded/unsafe | Not acceptable for protected writes |
| Read-only relaxed | read events may batch | Lower | bounded | Read-heavy paths |

## Required Dispatch Ordering

For protected writes in strict mode:

1. Receive query.
2. Evaluate policy.
3. Append policy event.
4. Append backend dispatch intent.
5. Ensure durability according to mode.
6. Forward to backend.
7. Append backend completion/failure.

For group commit mode, dispatch intent may be considered durable only when it falls within a configured crash window accepted by policy/customer.

## Crash Window

Group commit creates a bounded window where AXIS may crash after deciding but before fsync.

Required fields:

```json
{
  "durability_mode": "group_commit",
  "max_crash_window_ms": 50,
  "intent_enqueued_at": "...",
  "intent_fsynced_at": "..."
}
```

If the window is exceeded, AXIS must fail closed for protected writes.

## Suggested Defaults

| Environment | Mode | Max Crash Window |
|---|---|---:|
| Local lab | group_commit | 200ms |
| External reviewer demo | group_commit | 50ms |
| High-assurance pilot | strict_fsync or group_commit with acceptance | 0-50ms |
| Regulated production | customer-defined | contract-specific |

## Performance Components

| Component | Notes |
|---|---|
| Network hop | depends on topology |
| Protocol decode | small but not zero |
| SQL parse/classify | can dominate for complex SQL |
| Policy eval | depends on rules/catalog/context |
| Audit append | depends on WAL implementation |
| fsync | often dominates p99 |
| Approval store | required for APPROVAL_REQUIRED |
| Backend latency | not AXIS overhead but affects total |

## Backpressure and Circuit Breakers

AXIS must not queue infinitely.

Required thresholds:

- max audit queue depth;
- max policy queue depth;
- max outstanding backend dispatches;
- max active connections;
- max per-client risky requests/sec;
- max WAL disk usage percentage.

When thresholds exceed limits:

- reads may continue if safe;
- protected writes fail closed;
- readiness fails if sustained;
- operator alert emitted.

## Customer-Facing Performance Claim Rules

Do not state a number without:

- topology;
- workload;
- audit durability mode;
- policy complexity;
- PostgreSQL version;
- driver/protocol mode;
- p50/p95/p99;
- sample size;
- hardware.

Forbidden phrase:

```text
AXIS adds <2ms overhead.
```

Allowed pattern:

```text
In lab sidecar mode with Simple Query, local policy evaluation, and group-commit audit configured at 50ms, AXIS measured X/Y/Z p50/p95/p99 overhead under workload W.
```

Boring? Yes. Defensible? Also yes.

## Current Known Weaknesses

- Actual measurements are pending.
- fsync cost may be customer-storage dependent.
- group commit weakens immediate evidence durability.
- strict fsync may be unacceptable for low-latency OLTP.

## Success Looks Like

Customers choose a durability/performance mode knowingly, with explicit crash-window trade-offs.

## Failure Looks Like

AXIS silently switches to async audit for speed, then someone asks for proof after an incident. That sound is credibility leaving the building.
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# Approval Store HA and Consistency

## Purpose

This document defines high availability and consistency requirements for approval tickets in single-AXIS and multi-AXIS deployments.

Approval is an authorization mechanism. If the store lies, races, splits, or disappears, AXIS must fail safely.

## Store Requirements

The approval store must provide:

- durable writes;
- unique ticket IDs;
- compare-and-set resolution;
- immutable resolved state;
- idempotency key uniqueness;
- expiry based on store timestamp;
- operator identity recording;
- audit export capability;
- backup and restore procedure.

## Ticket Scope

Ticket idempotency key should include:

- normalized SQL fingerprint;
- target database;
- actor/service identity;
- tenant;
- environment;
- policy id/version;
- matched rule;
- risk level;
- parameter hashes where safe;
- transaction context;
- approval scope mode.

Do not approve raw SQL forever because one version was approved once. That is not approval; that is a loaded trap with paperwork.

## Resolution Model

States:

- pending;
- approved;
- rejected;
- expired;
- superseded;
- revoked;
- consumed, if single-use approval.

State transitions must be compare-and-set:

```text
pending -> approved
pending -> rejected
pending -> expired
approved -> consumed optional
approved -> revoked optional
```

Resolved tickets are immutable except for explicit revocation metadata.

## Multi-AXIS Consistency

In active/active mode:

- all AXIS nodes use the same approval store;
- no node may create local-only approvals;
- approval lookup and creation must be atomic;
- duplicate requests must return the same pending ticket where scope matches;
- approved ticket execution must re-evaluate policy version compatibility.

## Store Availability Modes

| Store State | Approval Policy Behavior |
|---|---|
| healthy | normal |
| degraded read-only | no new approvals; existing approved tickets may be accepted only if verified |
| unavailable | approval-required operations fail closed |
| partition suspected | fail closed or single-writer mode |
| stale replica | do not authorize |

## Approval Race Scenarios

### Same request repeated by same client

Return existing pending ticket if scope matches.

### Same request repeated by different client

Create separate ticket unless policy defines shared operational approval scope.

### Approved ticket retry

Retry must match approved scope and policy compatibility. If policy changed, re-evaluate.

### Two operators resolve same ticket

First compare-and-set wins. Second receives immutable-state error.

### Store clock skew

Use store authoritative time, not AXIS node time.

## Approval Consumption

Policies may define:

| Mode | Meaning |
|---|---|
| single_use | one successful retry consumes approval |
| time_window | retries allowed within window |
| actor_bound | only original actor may use |
| service_bound | service may retry |
| operator_bound | operator must execute via control plane |

Default for dangerous writes: single_use + actor/service bound.

## Audit Events

Required:

- `AXIS_APPROVAL_LOOKUP`
- `AXIS_APPROVAL_CREATED`
- `AXIS_APPROVAL_DEDUP_HIT`
- `AXIS_APPROVAL_RESOLVED`
- `AXIS_APPROVAL_CONSUMED`
- `AXIS_APPROVAL_STORE_UNAVAILABLE`
- `AXIS_APPROVAL_SCOPE_MISMATCH`
- `AXIS_APPROVAL_POLICY_VERSION_MISMATCH`

## Backup and Recovery

Approval store backup must preserve:

- pending tickets;
- resolved states;
- operator identities;
- timestamps;
- idempotency keys;
- policy versions;
- audit correlation IDs.

After restore, AXIS must not accidentally resurrect expired or consumed approvals.

## Current Known Weaknesses

- Store implementation is not selected in RFC v1.2.
- Strong consistency may add latency.
- Multi-region approval consistency is future work.

## Success Looks Like

Two AXIS nodes cannot independently approve the same dangerous request in conflicting ways.

## Failure Looks Like

The approval store partitions and both sides say “approved.” Distributed systems, always finding creative ways to make humans regret ambition.
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# Bypass Audit Gap and Reconciliation

## Purpose

This document defines how AXIS records, limits, and reconciles audit gaps when emergency bypass or degraded modes are used.

Bypass is operationally necessary, but it weakens AXIS evidence guarantees. This must be declared, not hidden under soothing language.

## Audit Gap Types

| Gap Type | Meaning |
|---|---|
| hard_bypass_gap | AXIS not inline; sees no traffic |
| audit_only_gap | AXIS inline but not enforcing |
| partial_visibility_gap | AXIS sees only some services |
| crash_gap | AXIS process died before completion audit |
| export_gap | evidence exists but was not exported before bypass |

## Hard Bypass Evidence Status

During hard bypass, AXIS cannot prove:

- queries were evaluated;
- dangerous writes were blocked;
- backend non-execution for blocked operations;
- approval requirements were enforced.

Evidence must shift to customer-provided sources:

- PostgreSQL native logs;
- pg_audit;
- application logs;
- connection pool logs;
- change management records;
- operator incident notes.

## Audit-Only Mode

Audit-only mode is a possible future degraded mode.

Requirements:

- AXIS remains inline.
- AXIS records queries and decisions.
- AXIS does not enforce selected policies.
- Every event includes `enforcement_mode="audit_only"`.
- UI and health endpoints clearly show degraded mode.
- Mode requires operator authorization and expiry.
- Audit-only mode cannot be silently enabled by config drift.

Audit-only mode should be used only when business continuity outweighs enforcement.

## Reconciliation Procedure

After bypass:

1. Record bypass start/end times.
2. Export AXIS pre-bypass bundle.
3. Collect PostgreSQL logs for window.
4. Collect application/pool logs.
5. Identify high-risk operations.
6. Compare operations against AXIS policy as offline analysis where possible.
7. Mark unverifiable operations.
8. Produce reconciliation report.
9. Append `AXIS_BYPASS_RECONCILIATION_COMPLETED` to audit WAL.
10. Review incident and update runbook.

## Reconciliation Report Schema

```json
{
  "schema_version": "axis.bypass.reconciliation.v1",
  "bypass_id": "bypass_...",
  "window_start": "...",
  "window_end": "...",
  "mode": "hard_bypass",
  "axis_audit_head_before": "...",
  "axis_audit_head_after": "...",
  "postgres_log_sources": ["..."],
  "operations_seen": 1234,
  "high_risk_operations": 4,
  "unverified_operations": 2,
  "manual_findings": [],
  "reviewed_by": "operator_id"
}
```

## Customer Contract Language

Customer-facing documents must state:

- hard bypass disables AXIS enforcement;
- hard bypass creates an AXIS evidence gap;
- AXIS can help reconcile but cannot prove what it did not observe;
- PostgreSQL logging must be enabled during bypass;
- emergency bypass must be time-bounded.

## Current Known Weaknesses

- Reconciliation cannot recreate AXIS non-execution proof.
- PostgreSQL logs may be insufficient without prior configuration.
- Audit-only mode is not implemented in PoC.
- Offline policy replay may not perfectly reconstruct runtime context.

## Success Looks Like

After bypass, everyone knows exactly what evidence was lost, what was reconstructed, and what remains unverified.

## Failure Looks Like

A bypass window disappears from the story because acknowledging it is uncomfortable. Discomfort is not a logging strategy.
# 30 — PoC Guardrails and Red Lines

## Purpose

This document defines the non-negotiable guardrails for the first AXIS Native PostgreSQL Wire Protocol laboratory PoC.

This PoC is not an enterprise pilot.
This PoC is not production-ready.
This PoC is not a general PostgreSQL proxy.
This PoC exists only to prove that AXIS can sit in the PostgreSQL wire path, intercept Simple Query traffic, evaluate policy before execution, enforce ALLOW/BLOCK/APPROVAL_REQUIRED decisions, and preserve audit evidence.

## Scope

The PoC scope is intentionally narrow:

- Accept a PostgreSQL client connection.
- Pass through startup/authentication flow in lab mode.
- Detect Simple Query messages.
- Extract SQL from Simple Query messages.
- Build an AXIS request envelope.
- Evaluate the SQL through the existing AXIS policy engine.
- Forward ALLOW queries to the backend PostgreSQL instance.
- Return PostgreSQL-compatible ErrorResponse for BLOCK decisions.
- Return PostgreSQL-compatible ErrorResponse for APPROVAL_REQUIRED decisions.
- Ensure BLOCK and APPROVAL_REQUIRED queries are never forwarded to backend PostgreSQL.
- Emit ordered audit events for every critical decision.
- Preserve fail-safe behavior under uncertainty.

## Non-Goals

The PoC must not implement:

- Extended Query Protocol.
- Parse / Bind / Execute / Sync support.
- Prepared statement state machine.
- Portals.
- COPY protocol.
- COPY FROM STDIN.
- COPY FROM PROGRAM.
- LISTEN / NOTIFY.
- Pipeline mode.
- Large objects.
- Replication protocol.
- CancelRequest production behavior.
- TLS termination.
- SCRAM production authentication.
- Connection pooling.
- PgBouncer replacement behavior.
- Load balancing.
- Query rewriting.
- SQL comment injection.
- Production HA.
- Multi-AXIS consistency.
- Approval store HA.
- Enterprise observability.

Unsupported protocol paths must fail closed.

## Red Lines

### 1. No monolithic proxy implementation

Do not place all logic inside a single `proxy.rs`.

The implementation must separate responsibilities:

- listener
- session lifecycle
- protocol message handling
- SQL interception
- backend forwarding
- client response generation
- policy envelope construction
- audit event emission
- state tracking

A monolithic proxy implementation is rejected.

### 2. No custom PostgreSQL protocol from scratch unless explicitly justified

Do not invent a PostgreSQL wire protocol parser from scratch.

Use a reviewed crate or a narrow, documented protocol decoder for the limited PoC scope.

If a crate is used, document:

- crate name
- version
- why it was selected
- what it handles
- what AXIS still handles manually
- known limitations

### 3. BLOCK must never reach backend

For BLOCK decisions:

- The original query must not be forwarded.
- No query byte from the blocked SQL may be written to the backend socket.
- Audit must record `backend_forwarded=false`.
- Audit must record `tcp_bytes_forwarded=0`.
- Client must receive PostgreSQL-compatible ErrorResponse.

PostgreSQL backend logs are not sufficient proof.

A byte-level backend mock or equivalent socket-level test must prove that blocked queries never reach backend.

### 4. APPROVAL_REQUIRED must never wait on the TCP socket

For APPROVAL_REQUIRED decisions:

- Do not keep the client socket waiting for human approval.
- Do not hold backend transaction locks.
- Do not forward the query to backend.
- Create approval ticket.
- Emit audit evidence.
- Return PostgreSQL-compatible ErrorResponse with machine-readable approval context.

Approval is out-of-band.

### 5. Unsupported protocol messages fail closed

The PoC must fail closed for unsupported protocol paths, including:

- Extended Query messages
- COPY
- FunctionCall
- replication startup
- pipeline mode
- unsupported CancelRequest behavior
- unknown message types

Fail closed means:

- do not forward unsafe or unknown traffic
- emit audit event
- return controlled PostgreSQL-compatible error where possible
- close connection if state becomes unsafe

### 6. Startup/auth pass-through is lab-only

Startup/auth pass-through in the PoC does not mean AXIS has verified client identity.

Any identity field derived from startup parameters must be treated as claimed and unauthenticated unless AXIS explicitly verifies it.

Policy enforcement must not rely on unauthenticated identity for high-trust decisions.

### 7. No query rewriting

AXIS must not rewrite user SQL.

ALLOW path must forward original query bytes.

Any AXIS-generated metadata command must be explicitly documented and must not silently alter user transaction semantics.

### 8. Transaction state must be explicit

AXIS must track at least:

- Idle
- InTransaction
- FailedTransaction
- PoisonedByAXIS
- ResetAttempted
- ResetConfirmed
- ExecutionUnknown
- Closed

If AXIS cannot determine safe state, it must fail closed.

### 9. Safety ROLLBACK is AXIS-generated and audit-recorded

If AXIS issues a safety ROLLBACK, it must be:

- explicitly AXIS-generated
- policy-exempt
- audit-recorded
- clearly distinguishable from client-generated ROLLBACK

### 10. Audit ordering must be deterministic

Audit events must follow a stable order:

1. query received
2. policy evaluated
3. decision applied
4. backend dispatch intent, only for ALLOW
5. backend dispatched, only after actual socket write
6. backend completed / failed / unknown
7. client response emitted

BLOCK and APPROVAL_REQUIRED must not emit backend dispatched events.

### 11. ErrorResponse must be PostgreSQL-compatible

AXIS-generated errors must include at least:

- Severity
- SQLSTATE
- Message
- Detail
- Hint where useful
- AXIS request ID
- approval ticket ID where applicable

ErrorResponse behavior must be tested against real PostgreSQL clients where possible.

### 12. Byte-level tests are mandatory

The PoC must include tests proving:

- ALLOW reaches backend.
- BLOCK does not reach backend.
- APPROVAL_REQUIRED does not reach backend.
- Unsupported protocol paths do not reach backend.
- Audit events are emitted in correct order.
- Backend down fails closed.
- Audit unavailable fails closed.
- Policy unavailable fails closed.

### 13. Existing AXIS guarantees must not regress

The PoC must not weaken:

- policy-before-execution
- audit evidence
- audit hash-chain continuity
- policy manifest integrity
- approval immutability
- fail-safe behavior
- chaos/failure-mode protections

### 14. This PoC cannot be marketed as production-ready

This PoC may only be described as:

“Native PostgreSQL Wire Simple Query laboratory proof of concept.”

It must not be described as:

- production-ready
- enterprise-ready
- drop-in replacement
- ORM-compatible
- complete PostgreSQL proxy
- PgBouncer alternative

## Acceptance Criteria

The PoC is accepted only if:

- psql or a controlled Simple Query client can connect.
- Safe Simple Query SELECT can pass through AXIS and return backend result.
- Risky write can be BLOCKED before backend dispatch.
- APPROVAL_REQUIRED creates ticket and returns controlled ErrorResponse.
- Byte-level backend mock proves zero backend bytes for BLOCK and APPROVAL_REQUIRED.
- Audit events include policy metadata.
- Unsupported protocol paths fail closed.
- Existing regression and chaos tests still pass.
- No production claims are introduced.

## Final Rule

If the implementation is unsure, it must not forward.

Uncertainty means BLOCK, audit, and controlled failure.# 31 - Simple Query PoC Results

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
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---


# CHANGELOG v1.2

## Added

- Production-realism hardening.
- Risk register.
- Restart recovery semantics.
- Approval idempotency model.
- Emergency bypass procedure.
- Operability and observability.
- RFC lifecycle and versioning.
- Extended Query roadmap.
- Cluster/failover/multi-AXIS consistency.
- Policy authoring guide.

## Strengthened

- Transaction behavior.
- ROLLBACK policy exemption.
- ErrorResponse field requirements.
- CommandComplete row-count audit.
- CancelRequest as production blocker.
- COPY FROM PROGRAM as critical blocked operation.
- Identity attribution distinction between claimed and verified identities.
- Performance targets made more honest.
- PgBouncer transaction pooling marked unsupported initially.

## Removed / Reframed

- Avoided “AXIS is not a proxy” language in native mode.
- Reframed as Security Enforcement Proxy / Deterministic Data Security Enforcement Point.
- Removed overconfident zero-change implication.
---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# CHANGELOG v1.2

## Summary

v1.2 hardens the AXIS Native PostgreSQL Integration RFC set after second-round external critique.

The focus is production realism:

- evidence format;
- CancelRequest design;
- performance/durability trade-offs;
- transaction model clarity;
- approval store HA;
- bypass audit gaps;
- observability;
- protocol fidelity;
- identity attribution;
- testing rigor.

## Added

- `25-AUDIT-WAL-FORMAT-AND-EVIDENCE-SPEC.md`
- `26-CANCELREQUEST-DESIGN.md`
- `27-PERFORMANCE-AND-DURABILITY-TRADEOFFS.md`
- `28-APPROVAL-STORE-HA-AND-CONSISTENCY.md`
- `29-BYPASS-AUDIT-GAP-AND-RECONCILIATION.md`

## Major Revisions

- `00-README.md`: expanded document map and v1.2 production gate.
- `04-TRANSACTION-STATE-MODEL.md`: clarified strict mode, future lenient mode, safety rollback exemption, ReadyForQuery behavior.
- `09-TEST-MATRIX.md`: added byte-level backend mock requirement and more protocol/failure tests.
- `10-CODEX-IMPLEMENTATION-BRIEF.md`: added crate evaluation, module boundaries, ErrorResponse field set, no-monolith warning.
- `11-PROTOCOL-FIDELITY-MATRIX.md`: added COPY detection, SQL feature matrix, CommandComplete audit.
- `12-IDENTITY-ATTRIBUTION-STRATEGY.md`: clarified unauthenticated claimed user, protected GUCs, RLS/service-account weakness.
- `13-THREAT-MODEL-AND-BYPASSES.md`: added COPY FROM PROGRAM, function side effects, timing side-channel, large query DoS.
- `14-PERFORMANCE-BUDGET.md`: replaced optimistic targets with measured-mode framing and realistic bands.
- `16-RISK-REGISTER.md`: expanded risk list and marked v1.2 additions.
- `19-EMERGENCY-BYPASS-PROCEDURE.md`: strengthened bypass modes and evidence gap handling.
- `20-OPERABILITY-OBSERVABILITY.md`: added metrics, traces, dashboards, high-cardinality guardrails.
- `23-CLUSTER-FAILOVER-AND-MULTI-AXIS.md`: strengthened policy consistency, failover modes, PgBouncer reset risk.

## Remaining Critical Open Risks

- Extended Query implementation remains required before real OLTP pilot.
- CancelRequest design exists but implementation is required before production pilot.
- Approval store technology not selected.
- External audit witness/signature strategy not finalized.
- Performance numbers are not yet measured.
- Lenient transaction mode remains future research.

## v1.2 Verdict

v1.2 is sufficient to guide a disciplined Simple Query lab PoC.

v1.2 is not sufficient to claim production readiness. Anyone claiming otherwise should be gently escorted away from deployment privileges.
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
# AXIS Native PostgreSQL Integration RFC v1.2 Package

This package contains the v1.2 RFC hardening set for AXIS Native PostgreSQL integration.

Place `docs/technical/native-pg-integration/` under the AXIS repository root.

This package is documentation source-of-truth, not generated PDF collateral.
# Technical Docs

This section is for AXIS implementation, operation, security, and protocol-level documentation.

Start here:

- `AXIS_INSTALL_GUIDE.md`
- `AXIS_ARCHITECTURE_OVERVIEW.md`
- `AXIS_SECURITY_MODEL.md`
- `AXIS_RUNBOOK.md`
- `AXIS_TEST_COMMANDS.md`
- `native-pg-integration/`

# AXIS Runtime Hardening

This document describes runtime guardrails added for bounded, deterministic AXIS operation under slow requests, bad requests, DB pressure, pool pressure, and abusive traffic.

## Runtime Profile

`AXIS_RUNTIME_PROFILE` accepts `local` or `production`.

Local mode is the default for developer workflows. Production mode fails fast unless `AXIS_OPERATOR_TOKEN` is configured with a strong non-default value. Production mode also defaults `AXIS_AUDIT_EXPORT_REQUIRES_OPERATOR=true`.

Invalid profile values fail startup. Weak production operator tokens such as short values, `changeme`, `default`, `password`, `secret`, or similar default-like values fail startup.

## Request Timeout

`AXIS_REQUEST_TIMEOUT_MS` defaults to `10000`.

Invalid, zero, or out-of-range values fail startup. The request timeout must be greater than `AXIS_DB_QUERY_TIMEOUT_MS` so database timeout handling can report DB execution state before the request wrapper expires.

When the listener request timeout expires, AXIS returns:

```json
{
  "error": {
    "code": "request_timeout",
    "message": "Request exceeded configured AXIS timeout.",
    "category": "runtime",
    "severity": "error",
    "safe_to_retry": false,
    "operator_action": "Check runtime saturation, upstream latency, and configured timeout. Retry only idempotent requests after reviewing request context.",
    "details": {
      "timeout_ms": 10000
    }
  }
}
```

## DB Timeout And Pool Pressure

`AXIS_DB_QUERY_TIMEOUT_MS` defaults to `8000`.
`AXIS_DB_CONNECT_TIMEOUT_MS` defaults to `5000`.
`AXIS_DB_POOL_MAX_CONNECTIONS` defaults to `10`.
`AXIS_DB_POOL_ACQUIRE_TIMEOUT_MS` defaults to `3000`.

Invalid values fail startup. Pool acquisition is bounded; pool exhaustion returns a structured error and does not panic. Query execution is bounded; if a query timeout occurs after a connection was acquired, AXIS reports `execution_state: "unknown"` and records execution-unknown audit evidence where a request context exists. Protected writes are not retried automatically.

Pool exhaustion:

```json
{
  "error": {
    "code": "db_pool_exhausted",
    "message": "Database connection pool is exhausted.",
    "category": "database",
    "severity": "critical",
    "safe_to_retry": false,
    "operator_action": "Check database health, pool limits, and request concurrency. Reduce traffic or increase pool size only after reviewing workload.",
    "details": {
      "acquire_timeout_ms": 3000
    }
  }
}
```

DB timeout:

```json
{
  "error": {
    "code": "db_timeout",
    "message": "Database operation exceeded configured timeout.",
    "category": "database",
    "severity": "security_critical",
    "safe_to_retry": false,
    "operator_action": "Check database health and query latency. Treat execution state as unknown when reported. Do not automatically retry protected writes.",
    "details": {
      "timeout_ms": 8000,
      "execution_state": "unknown"
    }
  }
}
```

When a bounded DB or enforcement failure is returned through an existing query-decision payload instead of a standalone envelope, the same structured metadata appears as `QueryResponse.error` and the legacy top-level `error_code` remains present. The embedded details are limited to safe fields such as reason, matched rule, policy metadata, SQL fingerprint, timeout values, execution state, session id, and prepared statement name.

## Body And SQL Size Limits

`AXIS_MAX_BODY_BYTES` defaults to `1048576`.
`AXIS_MAX_SQL_BYTES` defaults to `262144`.

No unlimited mode exists. Values above the compiled safe maximums fail startup. `AXIS_MAX_SQL_BYTES` must be less than or equal to `AXIS_MAX_BODY_BYTES`.

Oversized bodies return `request_body_too_large`. Oversized SQL returns `sql_too_large`. Raw oversized SQL is not included in the response.

## Rate Limiting

`AXIS_RATE_LIMIT_ENABLED` defaults to `true`.
`AXIS_RATE_LIMIT_REQUESTS_PER_MINUTE` defaults to `120`.
`AXIS_RATE_LIMIT_BURST` defaults to `30`.

The first enforced key for `POST /query` is actor. If actor is unavailable, AXIS uses a safe fallback bucket. `/health` is not rate-limited.

Rate-limited responses use HTTP 429:

```json
{
  "error": {
    "code": "rate_limited",
    "message": "Request rate limit exceeded.",
    "category": "runtime",
    "severity": "warning",
    "safe_to_retry": true,
    "operator_action": "Reduce request rate or adjust rate-limit configuration after reviewing traffic source.",
    "details": {
      "retry_after_seconds": 30
    }
  }
}
```

## Operator Auth

Operator-protected endpoints require `X-AXIS-Operator-Token` or `Authorization: Bearer <token>`.

Protected endpoints include policy mutation and approval resolution. Audit export is protected when `AXIS_AUDIT_EXPORT_REQUIRES_OPERATOR=true`; production profile enables that default. Local mode leaves export public by default because Evidence Bundle V1 is redacted and intended for auditor/customer transfer, but operators can enable auth for local or staging with the same variable.

Auth failures return:

```json
{
  "error": {
    "code": "operator_auth_required",
    "message": "Operator authorization is required.",
    "category": "auth",
    "severity": "error",
    "safe_to_retry": true,
    "operator_action": "Provide the configured operator credential through the approved server-side path. Do not expose tokens in browser code or logs.",
    "details": {}
  }
}
```

The Control Plane injects the operator token server-side through `/api/axis`; browser code does not receive the token.

## Runtime Metrics

`GET /runtime/metrics` returns safe operational counters, limits, and derived audit index status. It does not expose filesystem paths, raw SQL, private keys, operator token, backend URL, or environment secrets.

The stable top-level fields are:

- `status`
- `uptime_seconds`
- `requests`
- `decisions`
- `limits`
- `policy`
- `audit`
- `errors`

`errors` contains safe counters by code, category, and severity plus last-error timestamps. It does not contain stack traces, raw SQL, private paths, backend URLs, tokens, or database connection strings.

Health remains separate at `GET /health`.

## Validation Commands

Run:

```bash
cargo fmt
cargo check
cargo test
python scripts/axis_runtime_smoke.py --base http://localhost:6543
```

For Docker-backed validation, run the audit smoke, evidence verifier, regression, restart, chaos, and Control Plane real-mode checks from `docs/technical/AXIS_TEST_COMMANDS.md`.

## Stress Validation

Run the bounded local stress validator against a local AXIS runtime:

```bash
python scripts/axis_runtime_stress.py --base http://localhost:6543 --concurrency 25 --requests 500 --include-export --include-rate-limit
```

The script refuses non-local hosts. It exercises safe read concurrency, dangerous write floods, oversized SQL, malformed JSON, rate-limit pressure, DB pool pressure, audit verification under load, runtime metrics under load, and Evidence Bundle V1 export plus offline verification.

Expected success:

```text
AXIS Runtime Stress Validation: PASS
```

Any `FAIL` means AXIS returned an unsafe decision, leaked forbidden text, produced unstructured runtime failure, failed audit verification, or exported a bundle that the offline verifier rejected. Do not treat external review readiness as complete until the root cause is fixed or the environment blocker is recorded.

## Known Limitations

Audit visibility uses Audit Derived Index V1 for candidate selection when the index is ready. WAL remains canonical: returned event bodies and Evidence Bundle V1 exports are loaded from WAL offsets, and `/audit/verify` still verifies the WAL directly.

Decision traceability is exposed at `GET /audit/trace` and is read-only. It reconstructs safe trace sections from WAL-backed audit evidence, does not use runtime logs as proof, and does not execute SQL or mutate approval, policy, or audit state.

Request timeout cannot interrupt already-running synchronous filesystem operations. Audit WAL append remains fail-fast and fsync-backed; AXIS does not silently downgrade audit integrity to avoid latency.
# AXIS Secret Hygiene

AXIS Secret Hygiene & Key Identity Hardening keeps secrets out of routine runtime output and fails fast when production profile configuration uses local, demo, default, empty, or too-short credentials.

## Redaction Scope

Runtime error details, runtime log metadata, and evidence export metadata are passed through centralized redaction before they are returned or stored for operator review.

AXIS redacts these patterns at minimum:

- `Authorization: Bearer <token>` and standalone bearer tokens
- JWT-like compact tokens
- `AXIS_OPERATOR_TOKEN`
- `AXIS_JWT_HS256_SECRET`
- `password=...` connection-string fields
- `postgres://user:password@host/db`
- `secret=...`
- `token=...`
- `signing_key=...`
- `private_key=...`
- private-key PEM looking material

The replacement marker is:

```text
<REDACTED>
```

AXIS still returns useful diagnostics such as error codes, categories, safe reason codes, policy ids, request ids, limits, and audit verification state. It should not return raw credentials, bearer tokens, signing private keys, database URLs with passwords, raw authorization headers, or local `.env` contents.

## Weak Secret Rules

Production profile rejects weak/default secrets for operator tokens and JWT HS256 secrets.

Rejected examples include:

- empty values
- too-short values
- `change-me`
- `demo`
- `demo-secret`
- `test`
- `test-secret`
- `secret`
- `password`
- `123456`
- local/demo-labeled tokens such as `axis-enterprise-local-operator-token-000000000000`

When `AXIS_AUTH_MODE=jwt_hs256`, `AXIS_JWT_HS256_SECRET` is required. In `AXIS_RUNTIME_PROFILE=production`, it must be a strong non-default value.

`AXIS_RUNTIME_PROFILE=local` may use clearly labeled local-only demo secrets so reviewer and enterprise boundary validation remain runnable. Those values are not production credentials and are rejected if production profile is enabled.

## Production Expectations

Production deployments should set:

```text
AXIS_RUNTIME_PROFILE=production
AXIS_OPERATOR_TOKEN=<strong managed operator token>
```

If JWT HS256 is temporarily used:

```text
AXIS_AUTH_MODE=jwt_hs256
AXIS_JWT_REQUIRED=true
AXIS_JWT_HS256_SECRET=<strong managed local validation secret>
```

HS256 remains a local/demo authentication mode. Production identity should move to OIDC/JWKS, mTLS, cloud IAM, service identity, or equivalent trusted identity infrastructure.

## What Not To Commit

Never commit:

- `.env` files containing real values
- operator tokens
- JWT signing secrets
- PostgreSQL passwords or full database URLs with passwords
- evidence signing private keys
- KMS, Vault, or cloud secret-store credentials
- runtime audit/evidence data directories
- core dumps, crash dumps, or debug captures containing live memory

The repository may contain local demo placeholders and example values. They are intentionally labeled as local/demo and are not production credentials.

## Supplying Secrets

Provide secrets through deployment tooling, environment injection, or a managed secret store. Do not place real values in source files, Docker images, frontend environment variables, screenshots, tickets, or review reports.

Operator-protected endpoints accept `X-AXIS-Operator-Token` or `Authorization: Bearer <token>`. Browser code should not receive operator tokens; the control plane proxy is expected to inject server-side credentials.

## Validation

Run the Rust tests that cover redaction, weak-secret validation, operator auth errors, health output, and evidence signing metadata:

```bash
cargo test
```

For runtime validation, check that `/health`, `/runtime/metrics`, `/logs`, auth failures, and evidence exports do not include bearer tokens, `AXIS_OPERATOR_TOKEN`, `AXIS_JWT_HS256_SECRET`, database passwords, private keys, or full unredacted connection strings.
# AXIS Security Review Checklist

## Runtime Startup

- [ ] `AXIS_RUNTIME_PROFILE=production` fails without `AXIS_OPERATOR_TOKEN`.
- [ ] Weak production operator tokens fail startup.
- [ ] Startup fails when PostgreSQL is unreachable.
- [ ] Startup fails on invalid timeout, pool, body-size, SQL-size, or rate-limit settings.

## Config Validation

- [ ] `AXIS_MAX_SQL_BYTES` greater than `AXIS_MAX_BODY_BYTES` fails startup.
- [ ] `AXIS_REQUEST_TIMEOUT_MS` less than or equal to `AXIS_DB_QUERY_TIMEOUT_MS` fails startup.
- [ ] Invalid booleans and unsigned integer env vars fail startup.

## Policy Manifest Validation

- [ ] Missing manifest fails startup.
- [ ] Missing active policy file fails startup.
- [ ] Policy checksum mismatch fails startup.
- [ ] Policy version mismatch fails startup.
- [ ] Startup dry-run failure prevents serving traffic.

## Query Classification

- [ ] `SELECT` is classified as read.
- [ ] `DELETE` without `WHERE` is detected.
- [ ] Batch `UPDATE` is detected.
- [ ] DDL is detected.
- [ ] Prepared statement execution preserves resolved operation evidence.

## Policy Enforcement

- [ ] Safe reads are allowed by policy.
- [ ] Dangerous deletes are blocked or require approval, never unsafe `ALLOW`.
- [ ] Protected writes require durable decision evidence before DB execution.
- [ ] Audit commit failure before protected write blocks execution.

## Approval Immutability

- [ ] Approval creation records immutable request evidence.
- [ ] Resolution requires operator auth.
- [ ] Concurrent resolve attempts produce at most one final execution.
- [ ] Rejected approvals do not execute.

## Audit WAL Integrity

- [ ] WAL append is fsync-backed.
- [ ] Malformed WAL records are not silently skipped.
- [ ] Corrupt audit WAL fails safely.
- [ ] JSONL projection is not treated as canonical evidence.

## Hash-Chain Verification

- [ ] `/audit/verify` reports `verified` for an intact WAL.
- [ ] Tampering reports a non-verified status with location details.
- [ ] Verification does not expose backend filesystem paths.

## Evidence Bundle V1

- [ ] `/audit/export` returns `bundle_type=axis.evidence_bundle.v1`.
- [ ] Export includes policy id, policy version, and policy SHA-256.
- [ ] Export omits raw audit records and raw SQL fields.
- [ ] Offline verifier detects payload hash mismatch.

## Signing And Verifier Behavior

- [ ] Signing disabled exports use `signature_algorithm=none`.
- [ ] Signing enabled with invalid key material fails safely.
- [ ] Signed bundles verify only with the matching public key.
- [ ] Private signing keys are never included in responses or images.

## Timeout Behavior

- [ ] Request timeout returns structured `request_timeout`.
- [ ] DB timeout returns structured `db_timeout`.
- [ ] Timeout responses do not retry protected writes automatically.

## DB Timeout Behavior

- [ ] DB execution timeout reports `execution_state=unknown`.
- [ ] Execution-unknown evidence is written when a context exists.

## Pool Pressure Behavior

- [ ] Pool exhaustion returns structured `db_pool_exhausted`.
- [ ] Pool pressure does not panic the service.
- [ ] Health remains reachable after pool pressure.

## Rate Limiting

- [ ] Rate limiting can produce HTTP 429 under bounded abuse.
- [ ] 429 responses use `rate_limited`.
- [ ] `/health` remains available while query traffic is rate-limited.

## Operator Auth

- [ ] Policy mutation requires `X-AXIS-Operator-Token` or bearer token.
- [ ] Approval resolution requires operator auth.
- [ ] Audit export requires operator auth when configured.
- [ ] Auth errors do not echo valid or invalid tokens.

## Frontend Proxy Security

- [ ] Browser code calls `/api/axis/*`, not `localhost:6543`.
- [ ] `NEXT_PUBLIC_AXIS_API_BASE` is absent.
- [ ] `AXIS_BACKEND_URL` is server-only.
- [ ] Operator token injection occurs only in the Next.js route handler.

## Secret Redaction

- [ ] Responses do not contain `AXIS_OPERATOR_TOKEN`.
- [ ] Responses do not contain `AXIS_EVIDENCE_SIGNING_PRIVATE_KEY_B64`.
- [ ] Metrics and health responses do not expose credentials.

## Filesystem Path Redaction

- [ ] Runtime metrics do not expose `/app/`, `/var/lib/`, `C:\`, or host paths.
- [ ] Evidence exports mark filesystem paths redacted.
- [ ] Audit API errors do not expose backend paths.

## Docker Packaging

- [ ] Runtime container runs as non-root.
- [ ] Image does not copy local audit logs, `.env` files, `target/`, `node_modules/`, or `.git/`.
- [ ] Policy files are available read-only in Compose.
- [ ] Healthcheck reports backend health.
- [ ] Data volume behavior is documented.

## Known Limitations

- [ ] Reviewer understands local Compose is not HA production deployment.
- [ ] Reviewer understands Audit Derived Index V1 is derived, rebuildable, and non-authoritative; WAL remains canonical for verification and exported evidence.
- [ ] Reviewer understands unsigned bundles are not attributable to a signing key.
- [ ] Reviewer understands AXIS does not replace TLS, IAM, backups, or external monitoring.
