# AXIS Pilot Limitations And Next Steps

## No native PostgreSQL wire proxy in this version

This pilot does not implement native PostgreSQL wire protocol compatibility. AXIS is reached through HTTP `/query` for protected writes. It is not demonstrated here as a transparent PostgreSQL listener that arbitrary PostgreSQL clients can use without application changes.

## HTTP adapter integration model

The FastAPI backend uses an AXIS HTTP adapter. The application constructs normal SQLAlchemy ORM operations, the custom session compiles protected writes into SQL, and the adapter submits them to AXIS HTTP `/query`.

This is an application integration model, not a driver-level proxy model.

## SQLAlchemy routing/interception is v1, not transparent universal ORM support

`AxisRoutingSession` is a pilot integration. It covers the ORM and SQLAlchemy Core patterns used by the sample business app. It is not a complete SQLAlchemy dialect and has not been proven against arbitrary flush ordering, relationship cascades, custom type binders, stored procedures, server-generated non-primary values, or every bulk operation mode.

## Read/write split is intentional

The pilot intentionally sends safe reads directly to PostgreSQL and protected writes through AXIS. This keeps the demo focused on policy-controlled mutation paths. It does not prove universal inspection of every read query.

## Human approval does not keep database transactions open

The pilot does not keep PostgreSQL transactions, locks, or connection state open while waiting for a human approval. Holding transactions open for human workflows would be operationally fragile and is not the model demonstrated here.

## Approval uses rollback + explicit retry model

When AXIS returns `REQUIRE_APPROVAL`, the application rolls back local SQLAlchemy transaction state. An operator approves or rejects through AXIS. If approved, the original business operation is submitted again with `approval_id`.

This proves approval-controlled execution. It does not prove suspended in-place database transaction continuation.

## Control Plane visibility depends on currently implemented endpoints

The optional Control Plane profile can show AXIS state where supported by existing AXIS endpoints. If a Control Plane view is incomplete, reviewers should inspect the backend endpoints and evidence files directly.

## Audit evidence depends on existing AXIS audit APIs

The evidence package uses AXIS `/audit/events` and `/audit/verify`. The package does not independently recompute every hash-chain value outside AXIS. Independent offline verification can be added later.

## This pilot is not a production deployment guide

The compose stack, local operator token defaults, sample policy, sample database, and frontend are for reviewer validation. They are not a hardened production deployment recipe.

## Next step: reviewer feedback

Use reviewer feedback to identify which evidence items need stronger proof, which integration boundaries need clarification, and which production questions must be answered before a customer pilot.

## Next step: native wire feasibility research

Evaluate a native PostgreSQL wire-compatible path, including protocol parsing, authentication, prepared statements, transaction/session affinity, connection pooling, and compatibility with common PostgreSQL drivers.

## Next step: second stack demo, likely Java/Spring Boot + Hibernate

Build a second integration using a different enterprise stack, likely Java, Spring Boot, and Hibernate. The goal is to test whether AXIS policy, approval, rollback, and audit concepts transfer beyond Python and SQLAlchemy.

## Next step: pilot customer environment mapping

Map the integration model to real customer environments: database topology, ORM usage, CI/CD controls, policy ownership, approval workflow, audit retention, operational monitoring, and rollback requirements.
