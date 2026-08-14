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
