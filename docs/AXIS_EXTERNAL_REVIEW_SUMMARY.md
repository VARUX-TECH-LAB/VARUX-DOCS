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
- Client and upstream mTLS across the full write-path (native pgwire forwarder, CancelRequest path, and API DbPool), verified with certificate-verification-bypass mutation tests (invalid CA, hostname-only bypass, disabled verification) — all detected/killed, not just encryption without verification.
- Extended Query Protocol / ORM ground-truth compatibility verified end-to-end for Prisma (OID-less bind inference) and Hibernate + pgjdbc (explicit-OID binds, batch, savepoint/rollback), including positive and negative policy-enforcement controls with signed audit evidence.
- Optional operator token for mutating policy lifecycle endpoints.
- Docker Compose local PostgreSQL path.
- Backend regression, parser, approval, audit continuity, and policy lifecycle tests.

## What Is Not Implemented Yet

- Full RBAC or SSO.
- Some protocol-edge compatibility gaps remain under active hardening: unknown portal/statement handling, SQLSTATE redaction on certain error codes, named-portal re-Bind semantics, and DECLARE CURSOR support.
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
