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

- No RBAC.
- No production-grade tenant isolation.
- No external secrets management.
- No metrics or observability backend.
- No log rotation or retention policy.
- No full historical audit-chain verifier.
- No distributed audit-chain coordination across multiple AXIS instances.
- No production deployment hardening.
