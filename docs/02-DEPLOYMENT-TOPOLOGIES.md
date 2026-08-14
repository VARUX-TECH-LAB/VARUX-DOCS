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
