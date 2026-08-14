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
