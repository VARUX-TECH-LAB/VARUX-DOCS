---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---


# Approval Idempotency Model

## Purpose

This document defines how AXIS handles repeated APPROVAL_REQUIRED operations.

## Problem

Without idempotency, the same dangerous query can create many tickets, race approvals, and produce ambiguous authorization.

## Approval Scope

An approval must define exact scope:

```text
actor/service
tenant
database
environment
normalized_sql_hash
raw_sql_hash
policy_id
policy_version
risk_level
time_window
parameter_hashes_if_applicable
transaction_state
```

## Ticket Creation Rule

If an identical approval scope already has a pending ticket, AXIS should return the existing ticket unless policy requires one-ticket-per-attempt.

## Approval Reuse Rule

Approved tickets do not mean “always allow forever.”

Default:

- Approval authorizes one future matching execution attempt.
- Execution must re-evaluate policy.
- Policy version must match or ticket is invalid.
- Ticket expires.
- Ticket records consumed execution.

## Race Conditions

If two clients request the same dangerous operation:

- Same scope may deduplicate into one ticket.
- Different actor/service/tenant creates separate ticket.
- Approval consumption must be atomic.
- A consumed approval cannot be replayed.

## Rejection

Rejected tickets should not suppress future requests forever unless policy says so. Repeated rejected requests may trigger rate limiting.

## Audit Events

Required:

```text
AXIS_APPROVAL_TICKET_CREATED
AXIS_APPROVAL_TICKET_REUSED
AXIS_APPROVAL_TICKET_APPROVED
AXIS_APPROVAL_TICKET_REJECTED
AXIS_APPROVAL_TICKET_EXPIRED
AXIS_APPROVAL_TICKET_CONSUMED
AXIS_APPROVAL_SCOPE_MISMATCH
```

## Current Known Weaknesses

- Requires durable approval store.
- Requires concurrency-safe consumption.
- Extended Query parameters complicate scope.
- Approval UX must not encourage blind retries.

## Acceptance Criteria

A reviewer must be able to answer: what exactly was approved, by whom, for whom, under which policy, for how long, and whether it was used.
