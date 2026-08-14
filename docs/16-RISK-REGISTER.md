---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# Risk Register

## Purpose

This document tracks high-priority risks for AXIS Native PG mode. It must be updated at every major milestone.

## Severity Scale

| Severity | Meaning |
|---|---|
| Critical | Blocks production pilot or can cause unsafe execution/evidence failure |
| High | Major reliability/security concern requiring mitigation before broad pilot |
| Medium | Important but manageable with constraints |
| Low | Documented limitation or future improvement |

## Risk Register

| ID | Risk | Severity | Status | Mitigation |
|---|---|---:|---|---|
| R1 | Transaction divergence | Critical | Open | Strict rollback+close, future lenient design |
| R2 | Extended Query incompatibility | Critical | Open | EQ roadmap, driver matrix |
| R3 | CancelRequest unsupported | Critical | Open | Dedicated design in RFC 26 |
| R4 | Identity attribution loss | Critical | Open | Verified actor model, protected metadata |
| R5 | Audit WAL format unspecified | Critical | Addressed v1.2 | RFC 25 |
| R6 | Performance/durability contradiction | Critical | Addressed v1.2 | RFC 27 |
| R7 | Approval store single point of failure | Critical | Open | RFC 28 |
| R8 | Emergency bypass audit gap | Critical | Open | RFC 29 |
| R9 | Parser mismatch | Critical | Open | parser corpus, fail closed ambiguity |
| R10 | AXIS crash loses in-flight state | Critical | Open | restart recovery + durable dispatch intent |
| R11 | COPY FROM PROGRAM bypass | Critical | Open | grammar-level detection and block |
| R12 | Backend failover ambiguity | High | Open | primary health + manual/auto mode |
| R13 | PgBouncer transaction pooling mismatch | Critical | Open | unsupported; test rejection |
| R14 | Function side effects | High | Open | function allow/blocklist, schema awareness |
| R15 | Timing side-channel | Medium | Open | min latency/jitter/rate limit |
| R16 | ErrorResponse data leakage | High | Open | sanitized client errors |
| R17 | High-cardinality metrics overload | Medium | Open | observability guardrails |
| R18 | Audit WAL retention/disk growth | High | Open | retention/archive policy |
| R19 | Policy evaluation latency spike | High | Open | backpressure/fail-closed queue limits |
| R20 | Multi-AXIS clock skew | Medium | Open | monotonic store timestamps / expiry rules |
| R21 | ORM implicit prepared statements | Critical | Open | Extended Query support and compatibility matrix |
| R22 | Approval replay/dedup race | High | Open | RFC 18 + RFC 28 |
| R23 | RLS broken by service account | High | Open | protected GUC + customer DB review |
| R24 | Backend confirmed but client delivery unknown | High | Open | distinct audit event |
| R25 | Large query memory DoS | High | Open | pre-buffer message limits |
| R26 | Protected GUC tampering | High | Open | SET filtering |
| R27 | Cluster policy version drift | Critical | Open | policy version consistency gate |
| R28 | Shared approval store partition | Critical | Open | fail closed or single-writer guarantee |
| R29 | Audit-only bypass mode misunderstood as safe enforcement | Medium | Open | explicit UI/status mode |
| R30 | Sales overclaim | High | Open | customer-facing claims control |

## Required Review Cadence

- Before Simple Query PoC implementation.
- After Simple Query PoC passes tests.
- Before Extended Query implementation.
- Before external reviewer demo.
- Before customer pilot.
- After any incident or unexpected failure mode.

## Current Known Weaknesses

- Several Critical risks remain open.
- This risk register does not itself mitigate risk. Stunning, yes, but documents still do not execute code.

## Success Looks Like

No production pilot begins while Critical risks are marked open without explicit written exception.
