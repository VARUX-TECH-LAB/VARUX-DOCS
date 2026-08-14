---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# Audit WAL Format and Evidence Spec

## Purpose

This document specifies the AXIS audit WAL entry format, hash-chain semantics, canonicalization rules, export format, and verification procedure for Native PG mode.

If AXIS claims durable evidence, the evidence format must be explicit. “We logged it somewhere” is not evidence; it is a diary with infrastructure ambitions.

## Scope

This spec covers:

- WAL line format;
- required event fields;
- hash algorithm;
- canonical JSON rules;
- chain continuity;
- dispatch intent and completion events;
- backend non-forward proof fields;
- export bundle format;
- verification procedure;
- retention and external witnessing.

## WAL Record Format

Each WAL record is one canonical JSON object encoded as UTF-8 and terminated by `\n`.

No pretty printing. No comments. No trailing commas. Apparently this must be said.

Minimum envelope:

```json
{
  "schema_version": "axis.audit.wal.v1",
  "event_id": "evt_...",
  "event_type": "AXIS_POLICY_EVALUATED",
  "event_time_utc": "2026-05-21T15:30:00.000000Z",
  "instance_id": "axis-node-1",
  "session_id": "sess_...",
  "axis_request_id": "req_...",
  "sequence": 12345,
  "previous_event_hash": "hex...",
  "event_body": {},
  "event_hash": "hex..."
}
```

## Canonicalization

Hash input is canonical JSON with:

- UTF-8 encoding;
- sorted object keys;
- no insignificant whitespace;
- stable number representation;
- `event_hash` excluded from its own hash input;
- `previous_event_hash` included;
- arrays in original semantic order.

Canonical hash input:

```text
canonical_json(record_without_event_hash)
```

Hash algorithm:

```text
SHA-256(canonical_json(record_without_event_hash))
```

Encoding:

```text
lowercase hex
```

## Required Common Fields

| Field | Required | Meaning |
|---|---|---|
| `schema_version` | yes | WAL schema identifier |
| `event_id` | yes | globally unique event ID |
| `event_type` | yes | event taxonomy |
| `event_time_utc` | yes | timestamp from AXIS node |
| `instance_id` | yes | AXIS instance |
| `session_id` | conditional | connection/session ID |
| `axis_request_id` | conditional | request correlation |
| `sequence` | yes | local monotonic sequence |
| `previous_event_hash` | yes | prior WAL event hash or genesis marker |
| `event_body` | yes | event-specific payload |
| `event_hash` | yes | SHA-256 hash |

## Event Taxonomy

Required Native PG event types:

- `AXIS_CONNECTION_ACCEPTED`
- `AXIS_STARTUP_PASSTHROUGH`
- `AXIS_AUTH_PASSTHROUGH_OBSERVED`
- `AXIS_READY_FOR_QUERY_OBSERVED`
- `AXIS_QUERY_RECEIVED`
- `AXIS_POLICY_EVALUATED`
- `AXIS_BACKEND_DISPATCH_INTENT`
- `AXIS_BACKEND_DISPATCHED`
- `AXIS_BACKEND_COMPLETED`
- `AXIS_BACKEND_FAILED`
- `AXIS_BLOCK_APPLIED`
- `AXIS_APPROVAL_ISSUED`
- `AXIS_SAFETY_ROLLBACK_ISSUED`
- `AXIS_SAFETY_ROLLBACK_RESULT`
- `AXIS_EXECUTION_UNKNOWN`
- `AXIS_BACKEND_CONFIRMED_CLIENT_DELIVERY_UNKNOWN`
- `AXIS_CONNECTION_CLOSED`
- `AXIS_EMERGENCY_BYPASS_DECLARED`
- `AXIS_BYPASS_RECONCILIATION_COMPLETED`
- `AXIS_RESTART_RECOVERY_INCOMPLETE`

## Query Received Body

```json
{
  "sql_hash": "sha256:...",
  "sql_fingerprint": "...",
  "raw_sql_redacted": null,
  "query_mode": "simple",
  "message_len_bytes": 128,
  "transaction_state_before": "Idle",
  "unauthenticated_claimed_db_user": "app_user",
  "axis_actor_id": null,
  "client_addr_hash": "...",
  "target_database": "prod"
}
```

Raw SQL must not be stored by default unless explicitly enabled and redaction policy accepts it.

## Policy Evaluated Body

```json
{
  "decision": "BLOCK",
  "reason": "dangerous_delete_without_where",
  "risk": "CRITICAL",
  "matched_rule": "block-dangerous-delete",
  "policy_id": "prod-main",
  "policy_version": "prod_main@2026-05-01.1",
  "policy_sha256": "...",
  "approval_required": false,
  "backend_forward_allowed": false
}
```

## Dispatch Intent Body

Before forwarding protected execution, AXIS writes intent:

```json
{
  "dispatch_id": "dispatch_...",
  "axis_request_id": "req_...",
  "backend_addr": "hash-or-label",
  "bytes_planned": 128,
  "durability_mode": "fsync_per_critical_event|group_commit",
  "intent_durable": true
}
```

If AXIS crashes after intent but before completion, restart recovery can mark incomplete execution.

## Block Applied Body

```json
{
  "decision": "BLOCK",
  "backend_forwarded": false,
  "tcp_bytes_forwarded": 0,
  "proof_scope": "AXIS internal forwarder assertion",
  "independent_backend_witness": false,
  "error_sqlstate": "42501"
}
```

Important: `tcp_bytes_forwarded=0` is an AXIS-layer assertion. It is strengthened by byte-level tests and code structure, not magically witnessed by PostgreSQL.

## Backend Completed Body

```json
{
  "dispatch_id": "dispatch_...",
  "backend_confirmed": true,
  "command_tag": "UPDATE 12",
  "rows_affected": 12,
  "ready_for_query_status": "I",
  "client_delivery_confirmed": true
}
```

If backend completed but client delivery failed:

```json
{
  "event_type": "AXIS_BACKEND_CONFIRMED_CLIENT_DELIVERY_UNKNOWN",
  "event_body": {
    "backend_confirmed": true,
    "client_delivery_confirmed": false,
    "command_tag": "UPDATE 12",
    "operator_action": "application_may_retry_duplicate_operation"
  }
}
```

This is distinct from `AXIS_EXECUTION_UNKNOWN`.

## Execution Unknown Body

```json
{
  "dispatch_id": "dispatch_...",
  "bytes_written_to_backend": 128,
  "backend_response_received": false,
  "tcp_error": "connection_reset",
  "cannot_confirm_non_execution": true,
  "operator_action": "manual_backend_reconciliation_required"
}
```

## Export Bundle Format

An audit export bundle must include:

```text
bundle.json
wal-segment-000001.jsonl
wal-segment-000002.jsonl
MANIFEST_SHA256.json
SIGNATURE.json optional
```

`bundle.json`:

```json
{
  "bundle_schema": "axis.audit.bundle.v1",
  "export_id": "export_...",
  "created_at_utc": "...",
  "instance_id": "axis-node-1",
  "range_start_sequence": 1,
  "range_end_sequence": 1000,
  "range_start_hash": "...",
  "range_end_hash": "...",
  "policy_versions_seen": ["..."],
  "signature_algorithm": "none|ed25519",
  "external_witness": null
}
```

## Verification Procedure

Verifier must:

1. Parse every JSONL record.
2. Validate schema version.
3. Recompute each event hash.
4. Verify `previous_event_hash` continuity.
5. Verify sequence monotonicity.
6. Verify manifest file hashes.
7. Verify signature if present.
8. Report gaps, duplicates, invalid hashes, malformed records.

Verification output must be machine-readable.

## External Witnessing

Future production hardening should periodically publish Merkle roots or head hashes to an external witness:

- customer-controlled object store;
- timestamping service;
- signed transparency log;
- append-only external ledger.

This reduces the “you wrote logs after the fact” objection.

## Retention and Pruning

Audit WAL can grow quickly.

Required future retention policy:

- hot WAL retention window;
- archived bundle schedule;
- compression;
- integrity verification before deletion;
- legal hold support;
- disk low-watermark alert;
- retention per customer contract.

Never prune unexported WAL silently.

## Current Known Weaknesses

- External witness is future work.
- Signature scheme may be disabled in early PoC.
- Raw SQL storage is disabled by default, which may limit forensic readability.
- Per-instance chains require bundle-level correlation in multi-AXIS deployments.

## Success Looks Like

An auditor can verify that event N follows event N-1, that a blocked query was never dispatched by AXIS, and that the bundle was not quietly edited after export.

## Failure Looks Like

A compliance reviewer asks for the evidence format and receives “it’s JSON somewhere.” The room temperature drops five degrees.
