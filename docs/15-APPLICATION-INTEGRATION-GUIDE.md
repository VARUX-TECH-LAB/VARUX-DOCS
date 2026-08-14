---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---


# Application Integration Guide

## Purpose

This document explains what applications must expect when using AXIS Native PG mode.

## Honest Integration Claim

AXIS requires at minimum:

- connection string change.
- driver compatibility verification.
- protocol feature compatibility check.
- possible error-handling adaptation for approval workflows.
- TLS/certificate configuration in enterprise mode.

No claim of “zero application change” is allowed.

## Connection String

Example lab:

```text
postgres://user:pass@axis-host:6543/dbname?sslmode=disable
```

Production will differ.

## Approval Handling

APPROVAL_REQUIRED returns PostgreSQL ErrorResponse.

Applications should:

1. Detect AXIS approval condition through SQLSTATE and structured Detail.
2. Show or log approval ticket ID.
3. Avoid blind retry loops.
4. Retry only after approval and with idempotency guidance.
5. Discard connection if AXIS indicates connection poisoned.

## BLOCK Handling

BLOCK means policy denied execution before backend forward.

Applications should treat it as authorization failure, not database outage.

## Connection Poisoned

If AXIS returns a connection-poisoning error or closes the connection after transaction-internal BLOCK/APPROVAL, application pools must discard that connection.

Guide must include examples for:

- psycopg.
- node-postgres.
- JDBC/Hibernate.
- Npgsql.
- pgx.

## DDL and Migrations

DDL requiring approval should not be submitted inside large application transactions.

Migration tools such as Flyway, Liquibase, Alembic may wrap DDL in transactions. AXIS policy should either:

- require approval outside transaction.
- reject DDL-in-transaction with clear error.
- provide a migration workflow through HTTP/query or dedicated ops mode.

## ErrorResponse Fields

Applications should not parse ticket IDs out of free text if structured Detail is available.

AXIS should provide:

- SQLSTATE.
- Message.
- Detail with JSON-ish structured payload.
- Hint with human guidance.
- axis_request_id.

## Current Known Weaknesses

- ORM behavior varies.
- Simple Query PoC is not enough for most apps.
- Approval requires application awareness.
- Transaction reset behavior can trigger retries if app configuration is careless.

## Acceptance Criteria

The guide is acceptable only if a developer can understand what breaks, how to detect it, and when not to retry like a caffeinated raccoon.
