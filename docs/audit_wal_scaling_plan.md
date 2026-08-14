# Audit WAL Scaling And Derived Index

## Canonical Source Of Truth

The audit WAL remains the canonical source of evidence. The derived audit index is a read model only. It is safe to delete, rebuild, or ignore. `/audit/verify` verifies the WAL directly and does not use the index as proof.

Evidence Bundle V1 remains WAL-backed. `/audit/export` may use the index to choose candidate WAL offsets, but exported event bodies are re-read from WAL evidence before bundle hashing/signing.

## Audit Derived Index V1

AXIS writes the derived index to `AUDIT_INDEX_PATH`, defaulting to `./data/index/audit_index_v1.json`.

The persisted index has:

- metadata: `index_type=axis.audit_derived_index`, `index_version=1`, `built_at`, `source=audit_wal`, event counts, first/last IDs, last WAL hash, WAL byte length, WAL SHA-256, and `index_checksum`
- entries: `event_id`, `timestamp`, `event_type`, `decision`, `risk`, `actor`, `tenant`, `env`, `policy_id`, `approval_id`, `event_hash`, `previous_hash`, and `wal_offset`

The index intentionally does not store raw SQL, raw payloads, parameters, tokens, private keys, filesystem paths, or server environment values.

## Checksum Behavior

`index_checksum` is computed over deterministic canonical JSON with `metadata.index_checksum` removed. The checksum is therefore not self-referential.

The checksum protects the index payload from accidental or unauthorized edits. It does not make the index authoritative; WAL metadata and WAL-backed event loading still decide correctness.

## Build And Rebuild

Index rebuild is synchronous and derived from the WAL:

- verify WAL hash chain and event hashes first
- scan WAL records and collect safe searchable fields plus byte offsets
- compute WAL byte length and SHA-256
- compute index checksum excluding the checksum field
- persist the index atomically through a temporary file

Missing, corrupt, and stale indexes are rebuilt automatically when an audit read, export, or runtime metrics request needs the index.

## Missing, Stale, And Corrupt Indexes

Missing index:

- AXIS rebuilds from WAL when safe
- if rebuild cannot run safely, audit reads may fall back to strict WAL scan

Stale index:

- detected when WAL byte length, WAL SHA-256, or WAL last event hash differs from index metadata
- AXIS rebuilds from WAL when WAL verification passes

Corrupt index:

- detected by invalid JSON, invalid metadata, checksum mismatch, inconsistent event counts, or selected index entry mismatch against WAL
- AXIS discards the cached index state and rebuilds from WAL when safe

Malformed or hash-corrupt WAL:

- index is not marked ready
- AXIS does not silently repair the WAL
- `/audit/verify` continues to report WAL verification status directly
- audit APIs return structured WAL integrity errors instead of using index data as truth

## Pagination

Pagination cursors remain deterministic offset cursors: `v1:<matched_offset>`.

The index entries are stored in WAL order. Audit pages are produced newest-first by reversing that stable order, applying deterministic filters, and then applying the cursor offset. Repeated reads of the same index state produce the same next cursor.

Invalid cursors return a structured `invalid_pagination_cursor` error.

## Event Lookup

`GET /audit/events/:event_id` uses the index to find a candidate WAL offset by event ID or event hash. The returned full event is then loaded from the WAL at that offset. If the WAL event does not match the index entry, AXIS marks the index corrupt, rebuilds if safe, and otherwise falls back only when WAL reading remains safe.

`GET /audit/trace` is a read-only decision trace endpoint. It reconstructs traces from WAL-backed event bodies and does not treat runtime logs or index data as proof. The current implementation scans and verifies WAL evidence directly and keeps related events bounded by request `limit`.

## Filtering

`GET /audit/events` uses the index for candidate selection across actor, tenant, env, decision, risk, policy ID, approval ID, event type, and timestamp range. Returned summaries are loaded from WAL offsets and rechecked against the filters before response serialization.

## Export Selection

`GET /audit/export` uses the index only to select candidate event offsets. Evidence Bundle V1 events are built from WAL-loaded event details, not index-only bodies. Bundle `payload_sha256`, `event_hashes_sha256`, redaction flags, and optional Ed25519 signatures keep the existing V1 semantics.

If index/WAL candidate selection disagrees, WAL wins. AXIS marks the index corrupt, attempts a safe rebuild, or bypasses the index.

## Runtime Metrics

`GET /runtime/metrics` exposes safe index status:

```json
{
  "audit_index": {
    "status": "ready",
    "version": "1",
    "events_indexed": 123,
    "last_indexed_event_hash": "..."
  }
}
```

The status does not expose index paths, WAL paths, raw SQL, operator tokens, signing key material, or environment secrets.

## Operational Notes

- The index is disposable. Removing `AUDIT_INDEX_PATH` forces a rebuild on the next audit read/export/metrics call.
- WAL corruption remains a fail-fast startup condition through the audit logger.
- The active implementation keeps an in-memory cached index after safe validation and rebuilds when WAL bytes change.
- Hashing the WAL is still a linear byte read during validation/rebuild. This avoids trusting stale index metadata over WAL bytes, but it is not a substitute for future sealed-segment manifests.

## Known Limitations

- The index is a single JSON file. Very large deployments should move to sealed WAL segments plus segment indexes.
- Rebuild is synchronous in the request path today. A future operator job can rebuild asynchronously without changing WAL authority.
- The index does not provide retention, compaction, or archival semantics.
- `/audit/verify` remains a full WAL verification scan by design.
