---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# Approval Store HA and Consistency

## Purpose

This document defines high availability and consistency requirements for approval tickets in single-AXIS and multi-AXIS deployments.

Approval is an authorization mechanism. If the store lies, races, splits, or disappears, AXIS must fail safely.

## Store Requirements

The approval store must provide:

- durable writes;
- unique ticket IDs;
- compare-and-set resolution;
- immutable resolved state;
- idempotency key uniqueness;
- expiry based on store timestamp;
- operator identity recording;
- audit export capability;
- backup and restore procedure.

## Ticket Scope

Ticket idempotency key should include:

- normalized SQL fingerprint;
- target database;
- actor/service identity;
- tenant;
- environment;
- policy id/version;
- matched rule;
- risk level;
- parameter hashes where safe;
- transaction context;
- approval scope mode.

Do not approve raw SQL forever because one version was approved once. That is not approval; that is a loaded trap with paperwork.

## Resolution Model

States:

- pending;
- approved;
- rejected;
- expired;
- superseded;
- revoked;
- consumed, if single-use approval.

State transitions must be compare-and-set:

```text
pending -> approved
pending -> rejected
pending -> expired
approved -> consumed optional
approved -> revoked optional
```

Resolved tickets are immutable except for explicit revocation metadata.

## Multi-AXIS Consistency

In active/active mode:

- all AXIS nodes use the same approval store;
- no node may create local-only approvals;
- approval lookup and creation must be atomic;
- duplicate requests must return the same pending ticket where scope matches;
- approved ticket execution must re-evaluate policy version compatibility.

## Store Availability Modes

| Store State | Approval Policy Behavior |
|---|---|
| healthy | normal |
| degraded read-only | no new approvals; existing approved tickets may be accepted only if verified |
| unavailable | approval-required operations fail closed |
| partition suspected | fail closed or single-writer mode |
| stale replica | do not authorize |

## Approval Race Scenarios

### Same request repeated by same client

Return existing pending ticket if scope matches.

### Same request repeated by different client

Create separate ticket unless policy defines shared operational approval scope.

### Approved ticket retry

Retry must match approved scope and policy compatibility. If policy changed, re-evaluate.

### Two operators resolve same ticket

First compare-and-set wins. Second receives immutable-state error.

### Store clock skew

Use store authoritative time, not AXIS node time.

## Approval Consumption

Policies may define:

| Mode | Meaning |
|---|---|
| single_use | one successful retry consumes approval |
| time_window | retries allowed within window |
| actor_bound | only original actor may use |
| service_bound | service may retry |
| operator_bound | operator must execute via control plane |

Default for dangerous writes: single_use + actor/service bound.

## Audit Events

Required:

- `AXIS_APPROVAL_LOOKUP`
- `AXIS_APPROVAL_CREATED`
- `AXIS_APPROVAL_DEDUP_HIT`
- `AXIS_APPROVAL_RESOLVED`
- `AXIS_APPROVAL_CONSUMED`
- `AXIS_APPROVAL_STORE_UNAVAILABLE`
- `AXIS_APPROVAL_SCOPE_MISMATCH`
- `AXIS_APPROVAL_POLICY_VERSION_MISMATCH`

## Backup and Recovery

Approval store backup must preserve:

- pending tickets;
- resolved states;
- operator identities;
- timestamps;
- idempotency keys;
- policy versions;
- audit correlation IDs.

After restore, AXIS must not accidentally resurrect expired or consumed approvals.

## Current Known Weaknesses

- Store implementation is not selected in RFC v1.2.
- Strong consistency may add latency.
- Multi-region approval consistency is future work.

## Success Looks Like

Two AXIS nodes cannot independently approve the same dangerous request in conflicting ways.

## Failure Looks Like

The approval store partitions and both sides say “approved.” Distributed systems, always finding creative ways to make humans regret ambition.
