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
