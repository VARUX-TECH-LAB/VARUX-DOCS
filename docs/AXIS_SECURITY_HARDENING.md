# AXIS Security Hardening

This note describes the hardened enforcement and evidence paths implemented for the native PostgreSQL proxy.

## Parser and Enforcement Boundary

AXIS classifies SQL through PostgreSQL AST parsing and normalized AST traversal. Unsupported SQL, invalid UTF-8, raw NUL bytes, or unparseable extended-protocol bind values fail closed with `PARSE_REJECTED_UNTRUSTED_INPUT`.

Extended-protocol bind handling resolves typed parameters before classification. The resolver decodes PostgreSQL text and binary formats for supported OIDs, substitutes typed literals into the parsed template, and fingerprints the resolved template with a parameter hash so different bind values cannot share an enforcement decision accidentally. Raw parameter bytes are kept redacted and are not written to audit logs.

The evasion corpus in `tests/evasion_corpus/` covers comments, dollar strings, Unicode confusables, mixed encodings, multi-statement payloads, and extended-protocol parameter bypass attempts.

## Signed Audit WAL

Every audit WAL event has a monotonic `sequence_number`, Ed25519 `key_id`, event `signature`, `previous_hash`, and `event_hash`. Production startup requires `AXIS_AUDIT_SIGNING_KEY_PATH`; tests use ephemeral local keys. The adjacent `epochs.log` records key epochs so offline verification can validate signatures after key rotation.

Verification:

```powershell
cargo run --bin axis-audit-verify -- --epochs .\path\to\epochs.log .\path\to\audit.wal
```

## TLS and PostgreSQL Host Baseline

Client-to-proxy TLS is configured with:

- `AXIS_TLS_REQUIRE_CLIENT_CERT`
- `AXIS_TLS_CLIENT_CA_BUNDLE_PATH`
- `AXIS_TLS_SERVER_CERT_PATH`
- `AXIS_TLS_SERVER_KEY_PATH`

Production rejects `AXIS_TLS_REQUIRE_CLIENT_CERT=false`. Local mode allows cleartext only when TLS material is not configured.

Upstream PostgreSQL mTLS is configured with:

- `AXIS_TLS_UPSTREAM_CLIENT_CERT_PATH`
- `AXIS_TLS_UPSTREAM_CLIENT_KEY_PATH`
- `AXIS_TLS_UPSTREAM_CA_BUNDLE_PATH`

AXIS also checks the PostgreSQL `pg_hba_file_rules` view against the proxy-only hostssl/cert baseline. Relevant settings:

- `AXIS_PG_HBA_PROXY_CIDR`
- `AXIS_ENFORCE_PG_HBA_BASELINE`
- `AXIS_PG_HBA_CHECK_INTERVAL_MINUTES`

Generate the recommended rule:

```powershell
cargo run --bin axis-generate-pg-hba -- --db prod_main --role varux --proxy-cidr 10.2.0.0/24
```

When drift is detected after startup, AXIS writes a signed `AXIS_PG_HBA_DRIFT_DETECTED` audit event.

## Audit Checkpoint Anchoring

Checkpointing creates a compact signed statement of the latest durable audit WAL head. A checkpoint includes:

- schema `axis.audit.checkpoint.v1`
- WAL `sequence_number`
- WAL `event_hash`
- signing `key_id`
- timestamp
- Ed25519 signature over the checkpoint payload

Anchors are external delivery sinks. AXIS currently supports an append-only file replica and a webhook. Webhook receipts use schema `axis.audit.anchor_receipt.v1` and must echo the checkpoint id, sink id, and accepted status.

Configuration:

- `AXIS_AUDIT_CHECKPOINT_INTERVAL_EVENTS`
- `AXIS_AUDIT_CHECKPOINT_INTERVAL_MINUTES`
- `AXIS_AUDIT_CHECKPOINT_FAILURE_ALERT_THRESHOLD`
- `AXIS_AUDIT_CHECKPOINT_MAX_BACKOFF_SECONDS`
- `AXIS_AUDIT_CHECKPOINT_FILE_REPLICA_PATH`
- `AXIS_AUDIT_CHECKPOINT_WEBHOOK_URL`
- `AXIS_AUDIT_CHECKPOINT_WEBHOOK_BEARER_TOKEN`

Checkpoint anchor failures do not roll back a locally committed WAL event. Instead, AXIS retries the same unanchored checkpoint with bounded exponential backoff and writes signed `AXIS_CHECKPOINT_ANCHOR_FAILED` warning events. After the configured failure threshold, the warning is elevated as a hard alert in logs and payload.

Verification with checkpoints:

```powershell
cargo run --bin axis-audit-verify -- --epochs .\path\to\epochs.log --checkpoints .\path\to\checkpoints.log .\path\to\audit.wal
```

The verifier validates the WAL signature chain first, then rejects checkpoint records with bad signatures, missing WAL sequence numbers, non-monotonic checkpoint ordering, duplicate conflicting checkpoints, or checkpoint hashes that do not match the referenced WAL sequence.
