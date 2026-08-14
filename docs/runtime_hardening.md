# AXIS Runtime Hardening

This document describes runtime guardrails added for bounded, deterministic AXIS operation under slow requests, bad requests, DB pressure, pool pressure, and abusive traffic.

## Runtime Profile

`AXIS_RUNTIME_PROFILE` accepts `local` or `production`.

Local mode is the default for developer workflows. Production mode fails fast unless `AXIS_OPERATOR_TOKEN` is configured with a strong non-default value. Production mode also defaults `AXIS_AUDIT_EXPORT_REQUIRES_OPERATOR=true`.

Invalid profile values fail startup. Weak production operator tokens such as short values, `changeme`, `default`, `password`, `secret`, or similar default-like values fail startup.

## Request Timeout

`AXIS_REQUEST_TIMEOUT_MS` defaults to `10000`.

Invalid, zero, or out-of-range values fail startup. The request timeout must be greater than `AXIS_DB_QUERY_TIMEOUT_MS` so database timeout handling can report DB execution state before the request wrapper expires.

When the listener request timeout expires, AXIS returns:

```json
{
  "error": {
    "code": "request_timeout",
    "message": "Request exceeded configured AXIS timeout.",
    "category": "runtime",
    "severity": "error",
    "safe_to_retry": false,
    "operator_action": "Check runtime saturation, upstream latency, and configured timeout. Retry only idempotent requests after reviewing request context.",
    "details": {
      "timeout_ms": 10000
    }
  }
}
```

## DB Timeout And Pool Pressure

`AXIS_DB_QUERY_TIMEOUT_MS` defaults to `8000`.
`AXIS_DB_CONNECT_TIMEOUT_MS` defaults to `5000`.
`AXIS_DB_POOL_MAX_CONNECTIONS` defaults to `10`.
`AXIS_DB_POOL_ACQUIRE_TIMEOUT_MS` defaults to `3000`.

Invalid values fail startup. Pool acquisition is bounded; pool exhaustion returns a structured error and does not panic. Query execution is bounded; if a query timeout occurs after a connection was acquired, AXIS reports `execution_state: "unknown"` and records execution-unknown audit evidence where a request context exists. Protected writes are not retried automatically.

Pool exhaustion:

```json
{
  "error": {
    "code": "db_pool_exhausted",
    "message": "Database connection pool is exhausted.",
    "category": "database",
    "severity": "critical",
    "safe_to_retry": false,
    "operator_action": "Check database health, pool limits, and request concurrency. Reduce traffic or increase pool size only after reviewing workload.",
    "details": {
      "acquire_timeout_ms": 3000
    }
  }
}
```

DB timeout:

```json
{
  "error": {
    "code": "db_timeout",
    "message": "Database operation exceeded configured timeout.",
    "category": "database",
    "severity": "security_critical",
    "safe_to_retry": false,
    "operator_action": "Check database health and query latency. Treat execution state as unknown when reported. Do not automatically retry protected writes.",
    "details": {
      "timeout_ms": 8000,
      "execution_state": "unknown"
    }
  }
}
```

When a bounded DB or enforcement failure is returned through an existing query-decision payload instead of a standalone envelope, the same structured metadata appears as `QueryResponse.error` and the legacy top-level `error_code` remains present. The embedded details are limited to safe fields such as reason, matched rule, policy metadata, SQL fingerprint, timeout values, execution state, session id, and prepared statement name.

## Body And SQL Size Limits

`AXIS_MAX_BODY_BYTES` defaults to `1048576`.
`AXIS_MAX_SQL_BYTES` defaults to `262144`.

No unlimited mode exists. Values above the compiled safe maximums fail startup. `AXIS_MAX_SQL_BYTES` must be less than or equal to `AXIS_MAX_BODY_BYTES`.

Oversized bodies return `request_body_too_large`. Oversized SQL returns `sql_too_large`. Raw oversized SQL is not included in the response.

## Rate Limiting

`AXIS_RATE_LIMIT_ENABLED` defaults to `true`.
`AXIS_RATE_LIMIT_REQUESTS_PER_MINUTE` defaults to `120`.
`AXIS_RATE_LIMIT_BURST` defaults to `30`.

The first enforced key for `POST /query` is actor. If actor is unavailable, AXIS uses a safe fallback bucket. `/health` is not rate-limited.

Rate-limited responses use HTTP 429:

```json
{
  "error": {
    "code": "rate_limited",
    "message": "Request rate limit exceeded.",
    "category": "runtime",
    "severity": "warning",
    "safe_to_retry": true,
    "operator_action": "Reduce request rate or adjust rate-limit configuration after reviewing traffic source.",
    "details": {
      "retry_after_seconds": 30
    }
  }
}
```

## Operator Auth

Operator-protected endpoints require `X-AXIS-Operator-Token` or `Authorization: Bearer <token>`.

Protected endpoints include policy mutation and approval resolution. Audit export is protected when `AXIS_AUDIT_EXPORT_REQUIRES_OPERATOR=true`; production profile enables that default. Local mode leaves export public by default because Evidence Bundle V1 is redacted and intended for auditor/customer transfer, but operators can enable auth for local or staging with the same variable.

Auth failures return:

```json
{
  "error": {
    "code": "operator_auth_required",
    "message": "Operator authorization is required.",
    "category": "auth",
    "severity": "error",
    "safe_to_retry": true,
    "operator_action": "Provide the configured operator credential through the approved server-side path. Do not expose tokens in browser code or logs.",
    "details": {}
  }
}
```

The Control Plane injects the operator token server-side through `/api/axis`; browser code does not receive the token.

## Runtime Metrics

`GET /runtime/metrics` returns safe operational counters, limits, and derived audit index status. It does not expose filesystem paths, raw SQL, private keys, operator token, backend URL, or environment secrets.

The stable top-level fields are:

- `status`
- `uptime_seconds`
- `requests`
- `decisions`
- `limits`
- `policy`
- `audit`
- `errors`

`errors` contains safe counters by code, category, and severity plus last-error timestamps. It does not contain stack traces, raw SQL, private paths, backend URLs, tokens, or database connection strings.

Health remains separate at `GET /health`.

## Validation Commands

Run:

```bash
cargo fmt
cargo check
cargo test
python scripts/axis_runtime_smoke.py --base http://localhost:6543
```

For Docker-backed validation, run the audit smoke, evidence verifier, regression, restart, chaos, and Control Plane real-mode checks from `docs/technical/AXIS_TEST_COMMANDS.md`.

## Stress Validation

Run the bounded local stress validator against a local AXIS runtime:

```bash
python scripts/axis_runtime_stress.py --base http://localhost:6543 --concurrency 25 --requests 500 --include-export --include-rate-limit
```

The script refuses non-local hosts. It exercises safe read concurrency, dangerous write floods, oversized SQL, malformed JSON, rate-limit pressure, DB pool pressure, audit verification under load, runtime metrics under load, and Evidence Bundle V1 export plus offline verification.

Expected success:

```text
AXIS Runtime Stress Validation: PASS
```

Any `FAIL` means AXIS returned an unsafe decision, leaked forbidden text, produced unstructured runtime failure, failed audit verification, or exported a bundle that the offline verifier rejected. Do not treat external review readiness as complete until the root cause is fixed or the environment blocker is recorded.

## Known Limitations

Audit visibility uses Audit Derived Index V1 for candidate selection when the index is ready. WAL remains canonical: returned event bodies and Evidence Bundle V1 exports are loaded from WAL offsets, and `/audit/verify` still verifies the WAL directly.

Decision traceability is exposed at `GET /audit/trace` and is read-only. It reconstructs safe trace sections from WAL-backed audit evidence, does not use runtime logs as proof, and does not execute SQL or mutate approval, policy, or audit state.

Request timeout cannot interrupt already-running synchronous filesystem operations. Audit WAL append remains fail-fast and fsync-backed; AXIS does not silently downgrade audit integrity to avoid latency.
