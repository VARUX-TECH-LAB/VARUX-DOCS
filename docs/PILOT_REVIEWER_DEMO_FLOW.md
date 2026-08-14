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
