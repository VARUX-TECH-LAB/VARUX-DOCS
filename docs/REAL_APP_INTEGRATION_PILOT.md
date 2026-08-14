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
