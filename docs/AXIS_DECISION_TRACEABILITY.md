# AXIS Decision Traceability

## 1. Overview

AXIS decision traceability lets an operator inspect why AXIS made a recorded decision without re-executing SQL and without mutating security state.

The first endpoint is:

```text
GET /audit/trace
```

It reconstructs a bounded read-only trace from WAL-backed audit evidence and safe summaries.

## 2. What Decision Traceability Is

Decision traceability connects available evidence for a decision:

- request identity and `request_id`
- source audit event id/hash
- related audit events
- classifier output
- operation, target, scope, risk signals, and SQL fingerprint
- policy decision, final decision, reason code, matched rule, policy id/version/SHA-256
- approval id and approval resolution evidence when present
- execution state when present
- audit hash-chain position and verification status
- known limitations

## 3. What It Is Not

Decision traceability is not SQL replay, SQL re-execution, a database repair tool, an approval repair tool, or a policy mutation path.

It does not prove external business truth, database state outside AXIS, ticket correctness, or intent of an operator. It explains what AXIS recorded and can verify from its audit evidence.

## 4. Read-Only Guarantee

`GET /audit/trace` is read-only:

- it does not call the PostgreSQL executor
- it does not create approvals
- it does not resolve approvals
- it does not mutate policy state
- it does not append audit WAL events
- it does not write the JSONL projection
- it does not repair corrupted evidence

## 5. Source Of Truth Rules

The audit WAL is the source of truth. Runtime logs are operational visibility only. The derived audit index is disposable and is not proof.

The trace implementation scans and verifies WAL-backed audit events. It does not use runtime logs as evidence.

## 6. WAL-Backed Reconstruction

Trace reconstruction:

1. Validate that exactly one lookup key was provided.
2. Strictly scan the audit WAL.
3. Run existing hash-chain/event-hash verification.
4. Locate a source event.
5. Collect bounded related WAL events by matching `request_id`, `approval_id`, hash-chain neighborhood, and safe SQL fingerprint when available.
6. Build decision, classifier, policy, approval, execution, evidence, related-event, and limitation sections.

Unknown fields remain `null`, `unknown`, or empty arrays. AXIS does not fabricate missing classifier, policy, approval, or execution facts.

## 7. Derived Index Limitation

The derived audit index may be used by other audit APIs for candidate lookup. It is not authoritative evidence.

`GET /audit/trace` treats WAL records as authoritative and does not treat index contents as proof.

## 8. Runtime Logs Limitation

Runtime logs are not used to build decision traces. They can help operators find operational context, but they reset on restart, are bounded, and are not hash-chain evidence.

## 9. Endpoint Contract

```text
GET /audit/trace?event_hash=<sha256>
GET /audit/trace?event_id=<event_id>
GET /audit/trace?request_id=<request_id>
GET /audit/trace?approval_id=<approval_id>
```

Optional:

```text
limit=<1..100>
```

Default limit is `25`. The maximum is `100`.

Exactly one of `event_hash`, `event_id`, `request_id`, or `approval_id` is required.

## 10. Query Parameters

| Parameter | Meaning |
|---|---|
| `event_hash` | Locate a source event by 64-character SHA-256 audit event hash. |
| `event_id` | Locate a source event by audit event id. |
| `request_id` | Locate a decision trace for a request id. |
| `approval_id` | Locate approval-related decision evidence. |
| `limit` | Bound returned related events. Default `25`, maximum `100`. |

## 11. Response Example

```json
{
  "ok": true,
  "trace": {
    "trace_id": "trace_...",
    "source": {
      "lookup_type": "event_hash",
      "lookup_value": "...",
      "source_event_hash": "...",
      "source_event_id": "...",
      "request_id": "..."
    },
    "decision": {
      "decision": "BLOCK",
      "policy_decision": "BLOCK",
      "final_decision": "BLOCK",
      "reason_code": "policy_default_deny",
      "error_code": "policy_block",
      "risk": "CRITICAL"
    },
    "classification": {
      "operation": "DELETE",
      "query_type": "WRITE",
      "target": { "db": "prod_main", "schema": "public", "table": "orders" },
      "scope": "Batch",
      "risk_signals": ["delete_without_where"],
      "fingerprint": "..."
    },
    "policy": {
      "policy_id": "axis-prod-main",
      "policy_version": "prod_main@...",
      "policy_sha256": "...",
      "matched_rule": "default.fail_closed",
      "integrity": "recorded"
    },
    "approval": {
      "approval_id": null,
      "state": null,
      "resolved_by": null,
      "resolved_at": null,
      "decision": null
    },
    "execution": {
      "executed": false,
      "execution_state": "not_executed",
      "db_error_code": null
    },
    "evidence": {
      "event_hash": "...",
      "previous_hash": "...",
      "chain_position_known": true,
      "verification_status": "verified",
      "related_event_count": 3
    },
    "related_events": [],
    "limitations": [
      "Trace is reconstructed from WAL-backed audit evidence and safe summaries.",
      "Trace does not re-execute SQL.",
      "Trace does not prove external business truth."
    ]
  }
}
```

## 12. Error Behavior

Trace errors use the central AXIS structured error envelope.

| Condition | Code |
|---|---|
| Missing lookup key | `invalid_query_parameter` |
| Multiple lookup keys | `invalid_query_parameter` |
| Invalid limit | `invalid_query_parameter` |
| Malformed `event_hash` | `invalid_query_parameter` |
| Event/request not found | `audit_event_not_found` |
| Approval evidence not found | `approval_not_found` |
| Malformed WAL record | `audit_wal_corrupt` |
| Hash-chain/event-hash verification failure | `audit_verify_failed` |
| Unexpected internal failure | `internal_error` |

## 13. Security Redaction Rules

Trace responses do not include raw SQL, SQL parameters, operator tokens, DB URLs, backend URLs, private keys, env values, stack traces, raw WAL records, or private filesystem paths.

Trace includes SQL fingerprint and safe classifier/policy/approval summaries only.

## 14. Operator Use Cases

- Explain why a dangerous write was blocked.
- Confirm that an approval-required write did not execute at request time.
- Inspect rejection evidence after an approval was rejected.
- Distinguish DB timeout or execution-unknown evidence from ordinary policy block evidence.
- Link an audit event hash to related request, policy, approval, and execution records.

## 15. Known Limitations

- Trace reconstruction is only as complete as the recorded WAL evidence.
- Policy integrity in a trace means policy metadata was recorded in audit evidence; it does not revalidate historical policy bytes unless that evidence is separately verified.
- Related events are bounded by `limit`.
- Very large WAL files still require linear scan and verification.
- AXIS does not claim production readiness from this feature.

## 16. Test Commands

```powershell
cargo fmt --check
cargo check
cargo test trace_
curl.exe "http://localhost:6543/audit/trace?event_hash=<known_event_hash>"
curl.exe "http://localhost:6543/audit/trace?request_id=<known_request_id>"
curl.exe "http://localhost:6543/audit/trace?approval_id=<known_approval_id>"
cd control-plane
npm.cmd run typecheck
npm.cmd run lint
npm.cmd run build
npm.cmd run e2e:trace
```
