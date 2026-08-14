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
