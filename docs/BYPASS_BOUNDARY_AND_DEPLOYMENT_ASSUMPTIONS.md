# Bypass Boundary And Deployment Assumptions

The current pilot uses application-layer SQLAlchemy routing. Protected writes are routed through AXIS because the sample backend uses `AxisRoutingSession` and the AXIS HTTP `/query` adapter for protected write paths.

This is not the same as a network-level native PostgreSQL wire proxy.

## Current Integration Boundary

- Safe reads may go directly to PostgreSQL.
- Protected WRITE/DELETE/DDL paths are expected to go through AXIS by integration discipline.
- AXIS evaluates policy, returns structured outcomes, and emits audit evidence for protected operations.
- The HTTP adapter is the current integration model.

## What This Means

The pilot can demonstrate enforcement for code paths that use the integration correctly. It does not fully prove prevention of direct database write bypass by a misconfigured backend, leaked credential, or separate service with direct write access.

Direct DB write bypass prevention is not fully proven by this pilot. This is an explicit limitation, not a hidden claim.

## Production Hardening Would Require

- DB role separation.
- Credential control.
- Network policy.
- Restricted direct write credentials.
- Monitoring for direct database writes outside AXIS.
- Eventually native wire/proxy enforcement if transparent enforcement is required.

## Read/Write Split

The read/write split is intentional. It keeps low-risk read paths simple while routing protected mutations through AXIS. Reviewers should challenge whether this is operationally maintainable and what controls would be required to prevent accidental or intentional write bypass.
