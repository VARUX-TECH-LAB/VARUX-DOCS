# AXIS Operator Visibility Layer

AXIS v0.4 adds a read-only operator visibility layer over the existing enforcement core. The goal is to make runtime state, audit evidence, active policy, approvals, and evidence integrity inspectable without weakening the deterministic write-path controls.

## Why Visibility Matters

AXIS sits in front of production PostgreSQL and makes ALLOW, BLOCK, or APPROVAL_REQUIRED decisions before writes proceed. Operators need to answer four questions quickly:

- Is the service healthy?
- What policy is loaded?
- What evidence was recorded for recent decisions?
- Is the audit hash chain intact?

The v0.4 layer answers those questions with read-only endpoints and a control-plane UI backed by those endpoints.

## Backend Endpoints

### `GET /audit`

Returns recent audit events from the configured audit log path.

Query parameters:

- `limit`: optional event limit, default `100`, maximum `500`

Response fields:

- `ok`: request success flag
- `events`: recent audit events, newest first
- `count`: number of events returned
- `limit`: applied limit
- `malformed_count`: non-empty audit lines that were not JSON objects
- `order`: currently `newest_first`

If the audit log file does not exist, AXIS returns an empty event list.

The visibility response preserves fingerprints, normalized SQL, hashes, and event metadata. Raw SQL text in the embedded raw event view is redacted for operator safety.

### `GET /audit/:event_hash`

Fetches a single audit event by `event_hash`.

Responses:

- `200` with `{ "ok": true, "event": ... }` when found
- `404` when no audit event has that hash

Malformed audit lines are skipped during lookup.

### `GET /runtime/stats`

Returns dashboard-oriented runtime status:

- service name and status
- current timestamp
- uptime in seconds
- operating mode
- audit path, existence, event count, malformed count, and last event hash
- policy load status, version, path, and rule count
- pending approval count
- decision counts derived from audit events

Decision counts are derived from recorded audit decisions, not synthetic counters.

### `GET /policy`

Returns the active policy as loaded by the backend, plus:

- policy version
- policy path
- rule count
- operating mode

This endpoint is read-only. Policy editing is intentionally not enabled in v0.4.

### `GET /policy/status`

Returns a lightweight policy health summary:

- `loaded`
- `version`
- `path`
- `rules_count`

### `GET /evidence/verify`

Verifies audit evidence integrity without mutating or repairing the log.

Response fields:

- `valid`: whether verification passed
- `verification_depth`: `full_hash` when AXIS recomputed event hashes, `linkage_only` if a future fallback only checks links
- `checked_events`: events successfully verified before the first invalid record
- `malformed_count`: malformed records encountered
- `first_invalid_index`: zero-based event index for the first invalid record
- `first_invalid_line`: physical line number for the first invalid record
- `first_invalid_reason`: human-readable failure reason
- `last_event_hash`: last successfully verified event hash

The current implementation uses `full_hash` verification for AXIS audit events. It checks:

- each event has `event_hash`
- each event has `previous_hash`
- `previous_hash` links to the previous event hash
- recomputed canonical event hash matches `event_hash`

If the configured log contains legacy records without hash-chain fields, verification reports them honestly as invalid/malformed instead of hiding the issue.

## Control Plane Usage

The Next.js control plane consumes the v0.4 endpoints directly:

- Dashboard: `/health`, `/runtime/stats`, `/evidence/verify`
- Evidence Explorer: `/audit?limit=100`
- Policy Viewer: `/policy`, `/policy/status`
- Runtime: `/runtime/stats`, `/evidence/verify`
- Approval Center: existing `/approvals` and `/approvals/:approval_id/resolve`
- Query Console: existing `/query`

The UI remains an operator surface. It does not create fake production metrics when real endpoints are available.

## Intentionally Not Included

v0.4 does not add:

- policy editing
- multi-user auth or RBAC
- SaaS deployment management
- remote agent systems
- distributed audit-chain coordination
- log rotation or archival

Those are candidates for future versions, with policy editing and auth belonging behind stronger operator identity controls.
