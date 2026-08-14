---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---


# Policy and Audit Mapping

## Purpose

This document maps native PostgreSQL traffic into AXIS policy and audit concepts.

## AxisRequestEnvelope

Native PG mode must produce a normalized envelope for the existing policy engine.

Minimum fields:

```text
axis_request_id
session_id
connection_id
source_transport = pg_wire
client_addr
backend_addr
database
unauthenticated_claimed_db_user
verified_actor_id
service_name
tenant_id
application_name_client_claimed
application_name_axis_managed
query_mode
raw_sql_hash
sql_text_or_redacted_sql
normalized_sql
statement_kind
risk_classification
transaction_state
policy_id
policy_version
policy_sha256
received_at
```

## Identity Fields

Separate:

- `unauthenticated_claimed_db_user`: observed from client/startup/auth pass-through.
- `verified_actor_id`: identity AXIS verified through mTLS/OIDC/etc.
- `application_name_client_claimed`: client-provided value.
- `application_name_axis_managed`: AXIS-overwritten backend value.

Never collapse these fields. Compliance people love asking who verified what, because apparently evidence should mean things.

## Original Bytes and Metadata

User SQL must not be rewritten for correlation.

AXIS may issue separate backend session metadata commands, but those commands must be:

- Audit-recorded.
- Policy-classified as AXIS control statements.
- Protected from client overwrite.
- Not confused with user SQL.

## Audit Event Taxonomy

Required events:

```text
AXIS_CONNECTION_ACCEPTED
AXIS_BACKEND_CONNECTED
AXIS_STARTUP_PHASE_COMPLETED
AXIS_QUERY_RECEIVED
AXIS_POLICY_EVALUATED
AXIS_QUERY_FORWARDED
AXIS_BACKEND_COMMAND_COMPLETE
AXIS_BACKEND_RESPONSE_RECEIVED
AXIS_BLOCK_APPLIED
AXIS_APPROVAL_REQUIRED
AXIS_APPROVAL_TICKET_CREATED
AXIS_APPROVAL_TICKET_REUSED
AXIS_SAFETY_ROLLBACK_ISSUED
AXIS_TRANSACTION_STATE_CHANGED
AXIS_EXECUTION_UNKNOWN
AXIS_CLIENT_DELIVERY_UNKNOWN
AXIS_CONNECTION_POISONED
AXIS_CONNECTION_CLOSED
```

## Backend Did Not Receive Query

For BLOCK and APPROVAL_REQUIRED:

```json
{
  "backend_forwarded": false,
  "tcp_bytes_forwarded": 0,
  "forward_path_reachable": false,
  "decision_before_forward": true
}
```

This is an AXIS assertion. v1.2 tests must prove this with a backend mock or intercept layer, not only PostgreSQL query logs.

## CommandComplete Audit

When backend returns CommandComplete, AXIS should capture:

- command tag.
- row count if available.
- backend completion timestamp.
- transaction status after ReadyForQuery.
- backend response hash.

Example:

```json
{
  "event": "AXIS_BACKEND_COMMAND_COMPLETE",
  "command": "UPDATE",
  "row_count": 50000,
  "axis_request_id": "..."
}
```

## Execution Unknown vs Client Delivery Unknown

These are separate.

`EXECUTION_UNKNOWN`: AXIS forwarded to backend but cannot confirm backend completed.

`CLIENT_DELIVERY_UNKNOWN`: backend completed and AXIS observed completion, but client disconnected before receiving response.

These must not be merged. One is database uncertainty. The other is client delivery uncertainty.

## Parameter Handling

Raw bind parameters must not be written to audit by default. Use:

- redaction.
- hashing.
- allowlisted diagnostic capture.
- field-level sensitivity rules.

## Approval Linkage

Audit must answer:

- Who requested the operation?
- Which query caused approval?
- Who approved/rejected?
- Under which policy version?
- What exact execution was authorized?
- Did execution occur after approval or was a new evaluation required?

## Current Known Weaknesses

- Strong backend-side independent witnessing is not yet implemented.
- Parameter redaction rules need policy authoring support.
- CommandComplete row count extraction depends on protocol parsing.
- Approval idempotency is defined separately and must be implemented before enterprise pilot.

## Acceptance Criteria

An audit record is acceptable only if a reviewer can distinguish observed identity, verified identity, policy decision, backend reachability, execution confirmation, and delivery uncertainty.
