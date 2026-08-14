---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---


# Extended Query Roadmap

## Purpose

This document defines the path from Simple Query PoC to modern ORM compatibility.

## Why Extended Query Is Not Optional

Most serious production stacks use prepared statements and Extended Query flows. Simple Query PoC is transport validation, not production readiness.

## Phases

### Phase EQ-0: Discovery

- Test major drivers.
- Record protocol defaults.
- Identify simple-query downgrade options.
- Document which stacks cannot downgrade.

### Phase EQ-1: Parse Tracking

Support Parse messages:

- capture statement name.
- capture SQL template.
- classify template.
- store statement state per session.
- block dangerous templates.

### Phase EQ-2: Bind Tracking

Support Bind messages:

- capture portal.
- associate with statement.
- hash/redact parameter values.
- enforce parameter count/format.
- prevent raw sensitive audit leakage.

### Phase EQ-3: Execute

Support Execute:

- evaluate combined statement context.
- enforce policy before backend Execute.
- audit parameter hashes.
- handle portals and result streaming.

### Phase EQ-4: Sync and Error Recovery

- correct transaction status.
- correct ReadyForQuery handling.
- recover from failed statement state.
- Close statement/portal support.

### Phase EQ-5: Driver Compatibility

Test:

- psycopg3.
- asyncpg.
- node-postgres.
- Prisma.
- JDBC/Hibernate.
- Npgsql.
- Go pgx.
- SQLAlchemy.

## Policy Model

Policy may evaluate:

- SQL template.
- statement kind.
- parameter count.
- parameter type.
- parameter hashes.
- actor/tenant.
- transaction state.

Raw parameter values are not stored by default.

## Current Known Weaknesses

- Full semantic evaluation with parameters is complex.
- Prepared statement state can outlive assumptions.
- Portals and cursors introduce statefulness.
- Driver quirks will be annoying because software enjoys humiliation.

## Acceptance Criteria

Extended Query is accepted only when at least four major driver stacks pass ALLOW/BLOCK/APPROVAL/audit tests without unsafe downgrades.
