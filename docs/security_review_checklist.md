# AXIS Security Review Checklist

## Runtime Startup

- [ ] `AXIS_RUNTIME_PROFILE=production` fails without `AXIS_OPERATOR_TOKEN`.
- [ ] Weak production operator tokens fail startup.
- [ ] Startup fails when PostgreSQL is unreachable.
- [ ] Startup fails on invalid timeout, pool, body-size, SQL-size, or rate-limit settings.

## Config Validation

- [ ] `AXIS_MAX_SQL_BYTES` greater than `AXIS_MAX_BODY_BYTES` fails startup.
- [ ] `AXIS_REQUEST_TIMEOUT_MS` less than or equal to `AXIS_DB_QUERY_TIMEOUT_MS` fails startup.
- [ ] Invalid booleans and unsigned integer env vars fail startup.

## Policy Manifest Validation

- [ ] Missing manifest fails startup.
- [ ] Missing active policy file fails startup.
- [ ] Policy checksum mismatch fails startup.
- [ ] Policy version mismatch fails startup.
- [ ] Startup dry-run failure prevents serving traffic.

## Query Classification

- [ ] `SELECT` is classified as read.
- [ ] `DELETE` without `WHERE` is detected.
- [ ] Batch `UPDATE` is detected.
- [ ] DDL is detected.
- [ ] Prepared statement execution preserves resolved operation evidence.

## Policy Enforcement

- [ ] Safe reads are allowed by policy.
- [ ] Dangerous deletes are blocked or require approval, never unsafe `ALLOW`.
- [ ] Protected writes require durable decision evidence before DB execution.
- [ ] Audit commit failure before protected write blocks execution.

## Approval Immutability

- [ ] Approval creation records immutable request evidence.
- [ ] Resolution requires operator auth.
- [ ] Concurrent resolve attempts produce at most one final execution.
- [ ] Rejected approvals do not execute.

## Audit WAL Integrity

- [ ] WAL append is fsync-backed.
- [ ] Malformed WAL records are not silently skipped.
- [ ] Corrupt audit WAL fails safely.
- [ ] JSONL projection is not treated as canonical evidence.

## Hash-Chain Verification

- [ ] `/audit/verify` reports `verified` for an intact WAL.
- [ ] Tampering reports a non-verified status with location details.
- [ ] Verification does not expose backend filesystem paths.

## Evidence Bundle V1

- [ ] `/audit/export` returns `bundle_type=axis.evidence_bundle.v1`.
- [ ] Export includes policy id, policy version, and policy SHA-256.
- [ ] Export omits raw audit records and raw SQL fields.
- [ ] Offline verifier detects payload hash mismatch.

## Signing And Verifier Behavior

- [ ] Signing disabled exports use `signature_algorithm=none`.
- [ ] Signing enabled with invalid key material fails safely.
- [ ] Signed bundles verify only with the matching public key.
- [ ] Private signing keys are never included in responses or images.

## Timeout Behavior

- [ ] Request timeout returns structured `request_timeout`.
- [ ] DB timeout returns structured `db_timeout`.
- [ ] Timeout responses do not retry protected writes automatically.

## DB Timeout Behavior

- [ ] DB execution timeout reports `execution_state=unknown`.
- [ ] Execution-unknown evidence is written when a context exists.

## Pool Pressure Behavior

- [ ] Pool exhaustion returns structured `db_pool_exhausted`.
- [ ] Pool pressure does not panic the service.
- [ ] Health remains reachable after pool pressure.

## Rate Limiting

- [ ] Rate limiting can produce HTTP 429 under bounded abuse.
- [ ] 429 responses use `rate_limited`.
- [ ] `/health` remains available while query traffic is rate-limited.

## Operator Auth

- [ ] Policy mutation requires `X-AXIS-Operator-Token` or bearer token.
- [ ] Approval resolution requires operator auth.
- [ ] Audit export requires operator auth when configured.
- [ ] Auth errors do not echo valid or invalid tokens.

## Frontend Proxy Security

- [ ] Browser code calls `/api/axis/*`, not `localhost:6543`.
- [ ] `NEXT_PUBLIC_AXIS_API_BASE` is absent.
- [ ] `AXIS_BACKEND_URL` is server-only.
- [ ] Operator token injection occurs only in the Next.js route handler.

## Secret Redaction

- [ ] Responses do not contain `AXIS_OPERATOR_TOKEN`.
- [ ] Responses do not contain `AXIS_EVIDENCE_SIGNING_PRIVATE_KEY_B64`.
- [ ] Metrics and health responses do not expose credentials.

## Filesystem Path Redaction

- [ ] Runtime metrics do not expose `/app/`, `/var/lib/`, `C:\`, or host paths.
- [ ] Evidence exports mark filesystem paths redacted.
- [ ] Audit API errors do not expose backend paths.

## Docker Packaging

- [ ] Runtime container runs as non-root.
- [ ] Image does not copy local audit logs, `.env` files, `target/`, `node_modules/`, or `.git/`.
- [ ] Policy files are available read-only in Compose.
- [ ] Healthcheck reports backend health.
- [ ] Data volume behavior is documented.

## Known Limitations

- [ ] Reviewer understands local Compose is not HA production deployment.
- [ ] Reviewer understands Audit Derived Index V1 is derived, rebuildable, and non-authoritative; WAL remains canonical for verification and exported evidence.
- [ ] Reviewer understands unsigned bundles are not attributable to a signing key.
- [ ] Reviewer understands AXIS does not replace IAM, backups, or external monitoring.
