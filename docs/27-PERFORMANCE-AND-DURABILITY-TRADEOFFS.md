---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# Performance and Durability Trade-offs

## Purpose

This document defines the relationship between AXIS latency, audit durability, crash windows, and acceptable evidence guarantees.

Performance cannot be discussed honestly without durability mode. Any benchmark that omits audit fsync behavior is mostly decorative.

## Durability Modes

| Mode | Protected Write Forwarding Rule | Latency | Crash Window | Use |
|---|---|---:|---:|---|
| Strict Fsync | intent fsynced before forward | Highest | Minimal | High-assurance deployments |
| Group Commit | intent appended and fsynced within configured window | Medium | bounded | Early production candidate with customer acceptance |
| Async Audit | intent buffered only | Lowest | unbounded/unsafe | Not acceptable for protected writes |
| Read-only relaxed | read events may batch | Lower | bounded | Read-heavy paths |

## Required Dispatch Ordering

For protected writes in strict mode:

1. Receive query.
2. Evaluate policy.
3. Append policy event.
4. Append backend dispatch intent.
5. Ensure durability according to mode.
6. Forward to backend.
7. Append backend completion/failure.

For group commit mode, dispatch intent may be considered durable only when it falls within a configured crash window accepted by policy/customer.

## Crash Window

Group commit creates a bounded window where AXIS may crash after deciding but before fsync.

Required fields:

```json
{
  "durability_mode": "group_commit",
  "max_crash_window_ms": 50,
  "intent_enqueued_at": "...",
  "intent_fsynced_at": "..."
}
```

If the window is exceeded, AXIS must fail closed for protected writes.

## Suggested Defaults

| Environment | Mode | Max Crash Window |
|---|---|---:|
| Local lab | group_commit | 200ms |
| External reviewer demo | group_commit | 50ms |
| High-assurance pilot | strict_fsync or group_commit with acceptance | 0-50ms |
| Regulated production | customer-defined | contract-specific |

## Performance Components

| Component | Notes |
|---|---|
| Network hop | depends on topology |
| Protocol decode | small but not zero |
| SQL parse/classify | can dominate for complex SQL |
| Policy eval | depends on rules/catalog/context |
| Audit append | depends on WAL implementation |
| fsync | often dominates p99 |
| Approval store | required for APPROVAL_REQUIRED |
| Backend latency | not AXIS overhead but affects total |

## Backpressure and Circuit Breakers

AXIS must not queue infinitely.

Required thresholds:

- max audit queue depth;
- max policy queue depth;
- max outstanding backend dispatches;
- max active connections;
- max per-client risky requests/sec;
- max WAL disk usage percentage.

When thresholds exceed limits:

- reads may continue if safe;
- protected writes fail closed;
- readiness fails if sustained;
- operator alert emitted.

## Customer-Facing Performance Claim Rules

Do not state a number without:

- topology;
- workload;
- audit durability mode;
- policy complexity;
- PostgreSQL version;
- driver/protocol mode;
- p50/p95/p99;
- sample size;
- hardware.

Forbidden phrase:

```text
AXIS adds <2ms overhead.
```

Allowed pattern:

```text
In lab sidecar mode with Simple Query, local policy evaluation, and group-commit audit configured at 50ms, AXIS measured X/Y/Z p50/p95/p99 overhead under workload W.
```

Boring? Yes. Defensible? Also yes.

## Current Known Weaknesses

- Actual measurements are pending.
- fsync cost may be customer-storage dependent.
- group commit weakens immediate evidence durability.
- strict fsync may be unacceptable for low-latency OLTP.

## Success Looks Like

Customers choose a durability/performance mode knowingly, with explicit crash-window trade-offs.

## Failure Looks Like

AXIS silently switches to async audit for speed, then someone asks for proof after an incident. That sound is credibility leaving the building.
