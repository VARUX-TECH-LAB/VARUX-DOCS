---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# Performance Budget

## Purpose

This document defines early performance targets for AXIS Native PG mode and clarifies that performance numbers are meaningless unless tied to durability mode, policy complexity, backend distance, and driver protocol.

## Core Warning

A single p50 overhead target is not a production claim.

AXIS performance depends on:

- network hop count;
- policy evaluation cost;
- SQL parsing cost;
- audit WAL append mode;
- fsync/group commit strategy;
- approval store latency;
- backend response time;
- connection count;
- workload mix;
- transaction duration;
- driver protocol behavior.

Pretending otherwise is a fine way to create sales collateral and a poor way to keep systems alive.

## Measurement Modes

| Mode | Meaning |
|---|---|
| Lab loopback | client, AXIS, backend on same host/network |
| Local network | same LAN/VPC |
| Kubernetes sidecar | localhost pod path |
| Gateway service | network hop through AXIS service |
| HA/multi-instance | load balancer and shared stores included |

Every benchmark must name its mode.

## Early Target Bands

These are engineering targets, not customer SLAs.

| Path | Lab Target | Early Realistic Target |
|---|---:|---:|
| ALLOW Simple Query p50 overhead | 2-8ms | 8-25ms |
| ALLOW Simple Query p95 overhead | 5-20ms | 30-70ms |
| BLOCK p50 overhead | 2-10ms | 8-30ms |
| APPROVAL_REQUIRED p50 overhead | 5-30ms | 20-100ms depending store |
| ErrorResponse generation | <2ms | <10ms |
| Policy eval local p95 | <5ms | <20ms |
| Audit append p95 | depends on durability mode | depends on durability mode |

Hard customer claims must not be made until measured against the customer topology.

## Required Metrics

- p50/p95/p99 total AXIS overhead.
- policy evaluation latency histogram.
- SQL classification latency histogram.
- audit append latency histogram.
- audit fsync/group commit latency.
- backend connect latency.
- backend response latency.
- client response latency.
- approval store latency.
- active connections.
- queue depth/backpressure state.
- dropped/rejected connections.
- execution_unknown count.

## Benchmark Scenarios

1. Direct PostgreSQL baseline.
2. AXIS ALLOW with audit durability strict.
3. AXIS ALLOW with group commit durability.
4. AXIS BLOCK with no backend forward.
5. AXIS APPROVAL_REQUIRED with ticket creation.
6. Backend timeout.
7. Audit WAL pressure.
8. High connection count.
9. Risky query flood.
10. Large query rejection.

## Audit Durability Dependency

Performance claims must name audit mode:

| Audit Mode | Latency | Evidence Risk |
|---|---|---|
| fsync per critical event | Highest | Strongest local durability |
| group commit | Lower | bounded crash window |
| async append without sync | Lowest | unacceptable for protected writes unless explicitly accepted |

For protected production writes, AXIS must not forward before at least a durable dispatch intent under the selected durability contract.

See `27-PERFORMANCE-AND-DURABILITY-TRADEOFFS.md`.

## Backpressure

If policy or audit latency spikes, AXIS must apply backpressure instead of unbounded queuing.

Required behavior:

- per-client connection limits;
- global active session limit;
- audit queue high-water mark;
- policy queue high-water mark;
- risky query rate limits;
- fail-closed rejection under overload;
- operator-visible overload state.

## Current Known Weaknesses

- v1.2 does not yet include measured benchmarks.
- Extended Query will add state and classification cost.
- Approval store HA may add latency.
- TLS/mTLS adds handshake and CPU cost.
- fsync strategy can dominate tail latency.

## Success Looks Like

AXIS publishes numbers with context and trade-offs: topology, workload, audit mode, policy complexity, and tail latency.

## Failure Looks Like

Someone writes “<2ms overhead” on a slide and later discovers fsync, TLS, policy evaluation, network hops, and reality. Reality, rude as ever, wins.
