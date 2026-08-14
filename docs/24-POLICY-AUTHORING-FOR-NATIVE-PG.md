---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---


# Policy Authoring for Native PG Mode

## Purpose

This document defines policy authoring guidance for native PostgreSQL traffic.

## Why Native PG Policy Is Different

HTTP/query mode receives explicit request context. Native PG mode must infer context from protocol, session, identity, database, and SQL behavior.

Policy authors must not assume every SELECT is safe or every user string is verified.

## Required Policy Dimensions

Policies should support:

- environment.
- database.
- schema.
- table.
- operation kind.
- statement risk.
- actor/service.
- tenant.
- transaction state.
- query mode.
- policy version.
- function/procedure calls.
- COPY operations.
- SET/role/search_path commands.
- multi-statement behavior.
- approval requirements.

## Default Rules

Recommended defaults:

- Allow simple reads only if no write-like behavior.
- Block multi-statement by default.
- Block COPY by default.
- Block COPY FROM PROGRAM always.
- Block unsupported protocol states.
- Block DDL unless explicitly approved.
- Block SET ROLE unless explicitly allowed.
- Block protected GUC overwrite.
- Treat unknown functions as risky in production.
- Require approval for broad UPDATE/DELETE without safe predicate.
- Fail closed on parser uncertainty.

## Approval Policies

Approval policies must define:

- who may approve.
- approval scope.
- expiration.
- one-time vs reusable.
- whether transaction context invalidates approval.
- whether policy version changes invalidate approval.

## Testing Policies

Every native PG policy must have regression tests:

- ALLOW cases.
- BLOCK cases.
- APPROVAL cases.
- parser bypass corpus.
- transaction context cases.
- function/procedure cases.
- COPY cases.
- SET/search_path cases.

## Current Known Weaknesses

- Policy engine may need schema awareness.
- Function side effects are hard to classify.
- Extended Query parameters complicate authoring.
- RLS integration is future work.

## Acceptance Criteria

A native PG policy is acceptable only if it states what it allows, what it blocks, what it sends to approval, and what it does when it cannot classify.
