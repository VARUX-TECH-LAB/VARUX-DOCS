# AXIS Error Code Registry

## 1. Overview

AXIS uses one structured public error contract for backend API failures and Control Plane display. The registry is implemented in `src/errors.rs` and is intended to keep API responses, runtime logs, runtime metrics, operator guidance, and docs aligned.

AXIS remains a hardened local technical review candidate. This registry does not make enterprise production-readiness claims.

## 2. Error Response Contract

```json
{
  "error": {
    "code": "audit_commit_failed",
    "message": "Audit evidence could not be committed before protected write execution.",
    "category": "audit",
    "severity": "security_critical",
    "request_id": "req_or_uuid_when_available",
    "safe_to_retry": false,
    "operator_action": "Keep protected write traffic disabled. Check audit WAL availability, disk space, and write permissions. Run audit verification before restoring protected write traffic.",
    "details": {
      "reason": "safe public reason only"
    }
  }
}
```

`request_id` and `details` are optional. `safe_to_retry`, `operator_action`, `message`, `category`, `severity`, and `code` are always present in structured errors.

## 2.1 QueryResponse Embedded Error Metadata

The query API has two compatible structured-error forms:

- Request-level failures that cannot safely return a query decision, such as invalid JSON, oversized bodies, oversized SQL, request timeout, rate limit, and classifier/parser rejection, may return the standalone `{ "error": ... }` envelope.
- Enforcement decisions that already use the stable `QueryResponse` contract keep top-level fields such as `decision`, `policy_decision`, `reason_code`, `error_code`, `request_id`, `approval_id`, policy metadata, and audit behavior. When such a response is fail-closed or non-executed, AXIS also includes an optional `error` object with the same `AxisErrorBody` shape.

Example embedded query error:

```json
{
  "decision": "BLOCK",
  "policy_decision": "BLOCK",
  "reason_code": "policy_default_deny",
  "error_code": "policy_block",
  "error": {
    "code": "policy_block",
    "message": "Request was blocked by policy.",
    "category": "policy",
    "severity": "warning",
    "request_id": "76cf9f10-50f3-4b29-a536-8e0221501e71",
    "safe_to_retry": false,
    "operator_action": "Do not bypass AXIS. Review matched policy rule and audit evidence before changing policy.",
    "details": {
      "reason": "policy_default_deny",
      "matched_rule": "default.fail_closed",
      "decision": "BLOCK",
      "policy_decision": "BLOCK",
      "policy_id": "prod-main",
      "policy_version": "prod-main-v1",
      "sql_fingerprint": "..."
    }
  }
}
```

Clients should prefer `error` when present and retain `error_code` as a backwards-compatible fallback. The embedded object is generated from `AxisErrorCode`; listener code must not duplicate public messages or operator actions.

## 3. Category Model

Categories are: `request`, `sql`, `policy`, `approval`, `audit`, `audit_index`, `database`, `auth`, `runtime`, `config`, `evidence`, and `internal`.

## 4. Severity Model

Severities are: `info`, `warning`, `error`, `critical`, and `security_critical`.

`security_critical` is reserved for failures that can affect write-path safety, evidence integrity, approval execution safety, or policy lifecycle integrity.

## 5. HTTP Status Mapping

The canonical mapping lives in `src/errors.rs` on `AxisErrorCode::status()`.

| HTTP | Codes |
|---|---|
| 400 | `invalid_json`, `invalid_request_schema`, `invalid_query_parameter`, `missing_required_field`, `empty_sql`, `parser_error`, `parser_unsupported_syntax`, `unsupported_sql_shape`, `multi_statement_rejected`, `prepared_statement_requires_session`, `unresolved_prepared_statement`, `duplicate_prepared_statement`, `unsupported_prepared_statement`, `unsafe_read_shape`, `invalid_pagination_cursor` |
| 401 | `operator_auth_required` |
| 403 | `operator_auth_invalid`, `operator_auth_weak_config`, `policy_block`, `approval_rejected`, `approval_execution_blocked`, `evidence_signature_required` |
| 404 | `not_found`, `approval_not_found`, `audit_event_not_found` |
| 409 | `approval_already_resolved`, `policy_version_conflict`, `policy_activation_failed`, `policy_rollback_failed` |
| 413 | `request_body_too_large`, `sql_too_large` |
| 422 | `policy_validation_failed`, `policy_dry_run_failed`, `policy_manifest_missing`, `policy_manifest_invalid`, `policy_checksum_mismatch`, `policy_version_mismatch`, `evidence_bundle_invalid`, `evidence_signature_invalid` |
| 429 | `rate_limited` |
| 500 | `internal_error`, `approval_resolution_failed`, `evidence_export_failed`, `db_execution_failed`, `audit_verify_failed`, `config_invalid` |
| 503 | `db_unavailable`, `db_pool_exhausted`, `audit_unavailable`, `audit_commit_failed`, `audit_wal_corrupt`, `audit_index_corrupt`, `audit_index_rebuild_failed`, `approval_store_corrupt`, `runtime_unhealthy`, `policy_not_loaded`, `approval_audit_failed` |
| 504 | `request_timeout`, `db_timeout`, `execution_state_unknown` |

## 6. Full Error Code Table

| Code | Meaning | HTTP | Category | Severity | Retry | Reported to | Operator action |
|---|---|---:|---|---|---|---|---|
| `invalid_json` | Request body is not valid JSON. | 400 | request | error | yes | runtime_log, metrics | Correct client JSON. |
| `invalid_request_schema` | Request failed AXIS schema validation. | 400 | request | error | yes | runtime_log, metrics | Correct request fields. |
| `request_body_too_large` | Body exceeds configured limit. | 413 | request | warning | no | runtime_log, metrics | Reduce request size. |
| `sql_too_large` | SQL exceeds configured limit. | 413 | sql | warning | no | runtime_log, metrics, audit_for_query_rejection | Reduce SQL payload. |
| `invalid_pagination_cursor` | Cursor cannot be decoded. | 400 | request | error | yes | runtime_log, metrics | Refresh list cursor. |
| `invalid_query_parameter` | Query parameter is invalid. | 400 | request | error | yes | runtime_log, metrics | Correct query parameter. |
| `missing_required_field` | Required field is missing or empty. | 400 | request | error | yes | runtime_log, metrics | Send required field. |
| `empty_sql` | SQL is empty. | 400 | sql | error | yes | runtime_log, metrics, audit_for_query_rejection | Submit classifiable SQL. |
| `multi_statement_rejected` | More than one statement was submitted. | 400 | sql | warning | no | runtime_log, metrics, audit_for_query_rejection | Submit one statement. |
| `unsupported_sql_shape` | SQL shape is unsupported. | 400 | sql | error | yes | runtime_log, metrics, audit_for_query_rejection | Review SQL shape. |
| `parser_error` | SQL parser failed safely. | 400 | sql | error | yes | runtime_log, metrics, audit_for_query_rejection | Correct SQL syntax. |
| `parser_unsupported_syntax` | SQL syntax is unsupported by the AXIS parser. | 400 | sql | error | yes | runtime_log, metrics, audit_for_query_rejection | Treat as a parser coverage gap; add parser support before allowing. |
| `prepared_statement_requires_session` | PREPARE/EXECUTE/DEALLOCATE lacks `session_id`. | 400 | sql | error | yes | runtime_log, metrics, audit_for_query_rejection | Provide scoped session id. |
| `unresolved_prepared_statement` | EXECUTE cannot resolve statement in session. | 400 | sql | error | yes | runtime_log, metrics, audit_for_query_rejection | Refresh session state. |
| `duplicate_prepared_statement` | Statement name already exists in session. | 400 | sql | error | yes | runtime_log, metrics, audit_for_query_rejection | Deallocate or use another name. |
| `unsupported_prepared_statement` | Prepared command/name is unsupported. | 400 | sql | error | yes | runtime_log, metrics, audit_for_query_rejection | Use supported prepared lifecycle. |
| `unsafe_read_shape` | Read-like SQL has write-capable behavior. | 400 | sql | warning | no | runtime_log, metrics, audit_for_query_rejection | Treat as unsafe; rewrite SQL. |
| `policy_not_loaded` | No validated policy is loaded. | 503 | policy | critical | no | runtime_log, metrics, audit_if_startup_or_lifecycle | Disable write traffic until loaded. |
| `policy_manifest_missing` | Policy manifest is missing. | 422 | policy | error | no | runtime_log, metrics, audit_policy_lifecycle | Restore manifest. |
| `policy_manifest_invalid` | Manifest/schema/path is invalid. | 422 | policy | error | no | runtime_log, metrics, audit_policy_lifecycle | Verify manifest and policy bytes. |
| `policy_checksum_mismatch` | Manifest checksum does not match policy. | 422 | policy | security_critical | no | runtime_log, metrics, audit_policy_lifecycle | Do not serve this policy; verify SHA-256. |
| `policy_version_mismatch` | Manifest and policy versions disagree. | 422 | policy | error | no | runtime_log, metrics, audit_policy_lifecycle | Fix version metadata. |
| `policy_validation_failed` | Candidate policy failed validation. | 422 | policy | error | no | runtime_log, metrics | Fix validation errors. |
| `policy_dry_run_failed` | Policy activation dry-run failed. | 422 | policy | error | no | runtime_log, metrics, audit_policy_lifecycle | Fix unsafe dry-run findings. |
| `policy_activation_failed` | Activation failed. | 409 | policy | security_critical | no | runtime_log, metrics, audit_policy_lifecycle | Refresh lifecycle state and verify hashes. |
| `policy_rollback_failed` | Rollback failed. | 409 | policy | security_critical | no | runtime_log, metrics, audit_policy_lifecycle | Verify rollback target and hashes. |
| `policy_version_conflict` | Version status conflicts with operation. | 409 | policy | error | no | runtime_log, metrics | Refresh lifecycle state. |
| `policy_block` | Policy blocked request. | 403 | policy | warning | no | runtime_log, metrics, audit | Do not bypass; review matched rule. |
| `approval_not_found` | Approval id not found. | 404 | approval | info | yes | runtime_log, metrics | Refresh approval state. |
| `approval_already_resolved` | Approval was already resolved. | 409 | approval | error | yes | runtime_log, metrics, audit_if_resolution_attempt | Verify final decision in audit. |
| `approval_rejected` | Approval was rejected. | 403 | approval | warning | no | runtime_log, metrics, audit | Do not execute rejected request. |
| `approval_store_corrupt` | Approval store integrity failed. | 503 | approval | critical | no | runtime_log, metrics | Stop approval mutation; preserve store. |
| `approval_resolution_failed` | Approval resolution failed unexpectedly. | 500 | approval | error | no | runtime_log, metrics | Check runtime logs with request id. |
| `approval_execution_blocked` | Approved execution blocked by safety rule. | 403 | approval | security_critical | no | runtime_log, metrics, audit | Do not manually replay. |
| `approval_audit_failed` | Approval audit evidence failed. | 503 | approval | security_critical | no | runtime_log, metrics, audit | Restore WAL health before approvals. |
| `audit_commit_failed` | Audit evidence could not be committed. | 503 | audit | security_critical | no | runtime_log, metrics, audit_integrity_state | Disable protected writes; verify WAL. |
| `audit_unavailable` | Audit evidence cannot be read or written. | 503 | audit | critical | no | runtime_log, metrics | Restore WAL availability. |
| `audit_wal_corrupt` | WAL failed integrity checks. | 503 | audit | security_critical | no | runtime_log, metrics | Preserve WAL; run recovery runbook. |
| `audit_verify_failed` | Audit verification failed. | 500 | audit | critical | no | runtime_log, metrics | Verify files and recovery process. |
| `audit_index_corrupt` | Derived audit index is corrupt. | 503 | audit_index | critical | no | runtime_log, metrics | Treat index as disposable; verify WAL. |
| `audit_index_rebuild_failed` | Derived index rebuild failed. | 503 | audit_index | critical | no | runtime_log, metrics | Use WAL fallback after verification. |
| `audit_event_not_found` | Event id/hash not found. | 404 | audit | info | yes | runtime_log, metrics | Verify event identifier. |
| `evidence_bundle_invalid` | Evidence bundle is invalid. | 422 | evidence | error | no | runtime_log, metrics | Regenerate from verified WAL. |
| `evidence_export_failed` | Evidence export failed. | 500 | evidence | error | no | runtime_log, metrics | Verify WAL and filters. |
| `evidence_signature_invalid` | Evidence signature/config is invalid. | 422 | evidence | error | no | runtime_log, metrics | Verify signing public/private key config. |
| `evidence_signature_required` | Signature is required but absent. | 403 | evidence | error | no | runtime_log, metrics | Enable valid signing configuration. |
| `db_unavailable` | Database is unavailable. | 503 | database | critical | no | runtime_log, metrics | Check database connectivity. |
| `db_pool_exhausted` | DB pool could not provide connection. | 503 | database | critical | no | runtime_log, metrics, audit_for_write_path_failure | Reduce load or increase pool after review. |
| `db_timeout` | DB operation timed out. | 504 | database | security_critical | no | runtime_log, metrics, audit_execution_unknown | Treat execution state as unknown. |
| `db_execution_failed` | DB failed before confirmation. | 500 | database | error | no | runtime_log, metrics, audit_execution_failed | Confirm not forwarded before retry. |
| `execution_state_unknown` | Execution result is unknown. | 504 | database | security_critical | no | runtime_log, metrics, audit_execution_unknown | Reconcile database state before retry. |
| `operator_auth_required` | Operator auth is missing. | 401 | auth | error | yes | runtime_log, metrics | Provide configured auth path. |
| `operator_auth_invalid` | Operator auth is invalid. | 403 | auth | warning | yes | runtime_log, metrics, audit_for_sensitive_mutation | Verify token source; do not expose token. |
| `operator_auth_weak_config` | Operator auth config is weak. | 403 | auth | warning | no | runtime_log, metrics | Replace weak token config. |
| `request_timeout` | AXIS request timed out. | 504 | runtime | error | no | runtime_log, metrics | Check saturation and retry idempotent calls only. |
| `rate_limited` | Request rate exceeded. | 429 | runtime | warning | yes | runtime_log, metrics | Reduce rate or review limit config. |
| `config_invalid` | Runtime config is invalid. | 500 | config | error | no | runtime_log | Fix config without logging env values. |
| `runtime_unhealthy` | Runtime health is degraded. | 503 | runtime | critical | no | runtime_log, metrics | Check health, logs, WAL, and DB. |
| `not_found` | Resource or route not found. | 404 | runtime | info | yes | runtime_log, metrics | Verify identifier or endpoint. |
| `internal_error` | Internal fallback error. | 500 | internal | error | no | runtime_log, metrics | Capture request_id and sanitized logs. |

## 7. Reporting Destination Rules

Runtime logs receive structured summaries for meaningful API errors emitted through the central helpers, including embedded query-response errors. Runtime metrics count errors by code, category, and severity where the emitting path has `AppState`.

Audit evidence is intentionally narrower than runtime logs. AXIS writes audit evidence for security-relevant write-path and policy/approval lifecycle events, not for every client formatting problem.

## 8. Runtime Log Behavior

Runtime log entries may include timestamp, level, category, severity, `error_code`, `request_id`, endpoint, method, safe message, and operator action. Logs must not include raw SQL, stack traces, headers, secrets, private paths, DB URLs, or raw WAL records.

## 9. Audit Evidence Behavior

Audit-relevant examples include `policy_block`, `approval_rejected`, `audit_commit_failed`, `audit_wal_corrupt`, `policy_checksum_mismatch`, `policy_activation_failed`, `policy_rollback_failed`, `execution_state_unknown`, `db_timeout`, `approval_audit_failed`, and `approval_execution_blocked`.

Invalid JSON, ordinary bad query parameters, and other request-shape failures are runtime visibility only unless they pass through an existing security-relevant query rejection path.

`GET /audit/trace` uses the same structured envelope for trace lookup and WAL integrity failures. Missing or multiple lookup keys, malformed event hashes, and invalid limits return `invalid_query_parameter`. Missing event/request evidence returns `audit_event_not_found`; missing approval evidence returns `approval_not_found`. Malformed WAL records return `audit_wal_corrupt`; hash-chain or event-hash verification failures return `audit_verify_failed`.

## 10. Runtime Metrics Behavior

`/runtime/metrics` includes `errors.total`, `errors.by_code`, `errors.by_category`, `errors.by_severity`, `errors.last_error_at`, and `errors.last_security_critical_error_at`.

Known limitation: legacy helper paths that do not receive `AppState` can return structured errors without incrementing counters until those helpers are state-aware.

## 11. Operator Action Guide

Operator action text is defined per error code in `AxisErrorCode::operator_action()`. High-impact actions intentionally instruct operators to preserve WAL, disable protected write traffic when evidence is unsafe, avoid blind retries, and keep credentials out of logs and browsers.

## 12. Retry Guidance

`safe_to_retry=true` means the request can usually be retried after correcting client state, credentials, cursor, or rate. It does not mean AXIS will automatically retry protected writes.

`db_timeout`, `execution_state_unknown`, `audit_commit_failed`, approval audit failures, and policy lifecycle integrity failures are not safe to retry automatically.

## 13. Security Redaction Rules

Public error details may include only bounded safe fields such as `reason`, `limit_bytes`, `timeout_ms`, `retry_after_seconds`, `endpoint`, `method`, `policy_id`, `policy_version`, `policy_sha256`, `matched_rule`, `decision`, `policy_decision`, `risk`, `operation`, `query_type`, `scope`, `sql_fingerprint`, `session_id`, `prepared_statement_name`, `prepared_statement_command`, `request_id`, `approval_id`, `event_hash`, `cursor_format`, `safe_state`, `integrity_state`, and `execution_state`.

Public errors and Control Plane error views must not expose raw SQL, normalized SQL that reveals content, full request bodies, auth headers, tokens, env values, private paths, DB connection strings, stack traces, raw WAL records, or unbounded user-controlled strings.

## 14. Frontend Display Rules

The Control Plane validates the structured error body, displays code/message/category/severity/request_id/safe_to_retry/operator_action/safe details, and sanitizes details again client-side before rendering. It must not render backend URLs, tokens, raw SQL, stack traces, or private paths from error details.

## 15. Examples

Invalid JSON:

```json
{
  "error": {
    "code": "invalid_json",
    "message": "Request body must be valid JSON.",
    "category": "request",
    "severity": "error",
    "safe_to_retry": true,
    "operator_action": "Correct the client request shape. Do not include secrets or raw SQL in troubleshooting tickets."
  }
}
```

Database timeout:

```json
{
  "error": {
    "code": "db_timeout",
    "message": "Database operation exceeded configured timeout.",
    "category": "database",
    "severity": "security_critical",
    "request_id": "76cf9f10-50f3-4b29-a536-8e0221501e71",
    "safe_to_retry": false,
    "operator_action": "Check database health and query latency. Treat execution state as unknown when reported. Do not automatically retry protected writes.",
    "details": {
      "timeout_ms": 8000,
      "execution_state": "unknown"
    }
  }
}
```

## 16. Known Limitations

Some query decision failures still return the established query decision body with stable `reason_code`/`error_code` instead of the full structured error envelope to preserve ALLOW / BLOCK / REQUIRE_APPROVAL behavior and existing regression expectations.

Some policy, audit, and evidence helper paths produce structured errors without request IDs because no request context exists at that layer.

Runtime metrics count central state-aware error emissions. A future pass should move all helper-level errors to state-aware helpers to eliminate metric gaps and possible double counts where a runtime error log and structured response are emitted separately.
