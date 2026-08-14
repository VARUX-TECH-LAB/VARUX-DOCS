---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# Cluster, Failover, and Multi-AXIS Semantics

## Purpose

This document defines how AXIS behaves when multiple AXIS instances, backend failover, shared approval stores, and policy consistency enter the design.

Single-node assumptions must not leak into enterprise pilot architecture.

## Deployment Modes

| Mode | Status |
|---|---|
| Single AXIS lab | Supported for PoC |
| Single AXIS pilot | Acceptable only with explicit customer risk acceptance |
| Multi-AXIS active/passive | Candidate |
| Multi-AXIS active/active | Requires shared stores and consistency checks |
| Kubernetes sidecar per app | Candidate with local scope |
| Central gateway | Requires HA and drain model |

## Policy Version Consistency

Every AXIS instance must expose:

- active policy id;
- active policy version;
- policy sha256;
- manifest sha256;
- activation time;
- instance id.

Load balancers or readiness probes should remove instances whose policy version is stale.

Requests must audit policy version per decision. In multi-AXIS mode, a request cannot rely on “cluster policy” as a vague concept. Vague concepts do not survive subpoenas.

## Multi-AXIS Approval Consistency

Approval tickets must be globally consistent across AXIS instances.

Requirements:

- shared durable store or single-writer approval service;
- unique ticket ID generation;
- idempotency key uniqueness;
- approval scope hash;
- expiry based on store time, not instance wall clock;
- compare-and-set for resolution;
- immutable resolved state;
- audit correlation from all instances.

If approval store is partitioned, approval-required operations must fail closed or route to single available writer. Duplicate approvals for the same scope must not silently authorize duplicate executions.

See `28-APPROVAL-STORE-HA-AND-CONSISTENCY.md`.

## Clock Skew

Do not use individual AXIS node clocks as sole authority for approval expiry or policy activation in active/active mode.

Mitigations:

- use store-side timestamps;
- include monotonic version counters;
- tolerate bounded skew;
- record node timestamp and store timestamp separately.

## Backend Failover

v1.2 defines two modes.

### Manual Failover Mode

AXIS points to a configured primary endpoint. If backend becomes unavailable:

- new writes fail closed;
- reads may fail or route according to policy;
- operator updates backend endpoint;
- AXIS drains/reconnects;
- audit records backend failover event.

Manual mode is simpler and safer for early pilots.

### Automatic Failover Mode

AXIS uses a resolver/service discovery mechanism to identify the current primary.

Requirements:

- primary health check;
- write target verification;
- read-only detection;
- stale primary prevention;
- split-brain handling;
- audit failover event;
- operator alert.

Automatic failover must not route writes to a replica because “it was reachable.” Reachability is not authority. This remains difficult for humans, load balancers, and apparently civilization.

## Read Replica Behavior

If AXIS supports read routing in future:

- policy must classify read vs write safely;
- transaction state must remain consistent;
- reads after writes may require primary;
- replica lag must be visible;
- SELECT functions with side effects must not be routed as harmless reads.

Read routing is not part of Simple Query PoC.

## PgBouncer Compatibility

| PgBouncer Mode | v1.2 Status |
|---|---|
| No PgBouncer | Lab supported |
| PgBouncer behind AXIS, session pooling | Candidate with tests |
| PgBouncer behind AXIS, transaction pooling | Unsupported |
| PgBouncer before AXIS | Not recommended |

### Session Pooling Reset Risk

PgBouncer session pooling can run `server_reset_query` such as `DISCARD ALL` or `RESET ALL` when returning backend connections.

AXIS must know whether reset commands occur and how they affect:

- prepared statements;
- protected GUCs;
- application_name;
- axis_request_id correlation;
- transaction state;
- backend session identity.

This must be tested before PgBouncer session pooling is accepted.

## Audit Chain in Multi-AXIS

Options:

| Option | Pros | Cons |
|---|---|---|
| Per-instance audit chain | Simple, local integrity | Requires bundle correlation |
| Central audit writer | Easier global order | New bottleneck/SPOF |
| Per-instance + external witness | Scalable and verifiable | More complex |

v1.2 recommendation:

- keep per-instance WAL chains;
- periodically export Merkle roots/external witness entries;
- correlate via cluster event index.

## Current Known Weaknesses

- Automatic backend failover is not implemented.
- Active/active approval consistency requires a real shared store design.
- Multi-instance global audit ordering is not solved by local hash chains.
- PgBouncer transaction pooling is unsupported.

## Success Looks Like

A query decision remains explainable even when traffic crosses multiple AXIS instances, backend primary changes, and approval state is shared.

## Failure Looks Like

Two AXIS nodes approve the same dangerous operation differently because one had yesterday’s policy and the other had today’s optimism.
