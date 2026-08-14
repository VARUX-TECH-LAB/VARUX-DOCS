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
