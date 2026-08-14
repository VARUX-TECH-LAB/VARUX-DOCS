---
status: Accepted and implemented
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-07-10
source_of_truth: repository markdown and src/pgwire/cancel.rs
supersedes: 26-CANCELREQUEST-DESIGN.md implementation status
---

# RFC-032: CancelRequest and Mid-Query Backend Cancellation

## Decision

AXIS supports PostgreSQL out-of-band `CancelRequest` without disclosing the
backend process ID or secret key to the client. The backend `BackendKeyData`
message is intercepted during startup, replaced with an AXIS-generated key,
and stored in a process-local registry until the owning client session closes
or the 24-hour safety TTL expires.

Possession of the AXIS-generated key is the MVP authorization mechanism. A
future identity-aware policy hook belongs after registry resolution and before
backend forwarding. The current CancelRequest connection has no independently
authenticated user or role, so it must not infer identity from its source IP.

## Protocol Flow

1. PostgreSQL sends `BackendKeyData(real_pid, real_secret)` on the session
   connection.
2. AXIS generates a cryptographically random `(proxy_pid, proxy_secret)` with
   the operating system RNG.
3. AXIS stores the proxy key, real key, backend address, target session ID,
   creation/expiry state, and duplicate-suppression state in the shared pgwire
   services registry.
4. AXIS sends only `BackendKeyData(proxy_pid, proxy_secret)` to the client.
5. The client opens a new TCP connection and sends a 16-byte CancelRequest with
   the proxy key.
6. AXIS resolves the key, opens a new connection to the recorded backend, and
   writes a CancelRequest containing the real key. PostgreSQL sends no response;
   AXIS closes both cancel connections without returning client bytes.

## Security and Abuse Controls

- Real backend process IDs and secrets are never sent to clients or audit logs.
- Proxy secrets are never logged. Audit uses the proxy process ID and a
  96-bit displayed SHA-256 fingerprint of the complete proxy key.
- Unknown, expired, closed-session, and rate-limited keys all receive the same
  wire behavior: no response and connection close.
- Cancel handling has a 100 ms minimum response envelope, including a 75 ms
  backend connect/write timeout. This reduces practical key-existence timing
  disclosure but is not a cryptographic constant-time guarantee under host
  scheduling or network stalls.
- A process-local fixed-window limit allows 128 cancel attempts per source IP
  per second. Excess attempts are silently rejected and audited.
- Repeated requests for one key inside 250 ms are audited and suppressed. The
  key remains usable after that window so a later query on the same PostgreSQL
  session can also be cancelled.

## Lifecycle and Race Semantics

The normal pgwire session owns its registry registration. `Drop` marks the
entry inactive and removes it on every session exit path, including startup
failure and client disconnect. Expired entries are purged on registration and
lookup.

Registry resolution and session teardown have an explicit linearization:

- If teardown removes the entry first, cancel is rejected silently.
- If lookup reserves the cancel first, at most one already in-flight cancel
  connection may reach PostgreSQL while the session is closing. PostgreSQL
  treats cancellation of a closed or no-longer-running backend as a no-op.

The registry is memory-only. AXIS restart invalidates all proxy keys. A driver
must reconnect before it can cancel a query through the restarted proxy.

## Audit Contract

The implementation emits:

- `AXIS_BACKEND_KEYDATA_INTERCEPTED`
- `AXIS_BACKEND_KEYDATA_SUBSTITUTED`
- `AXIS_CANCEL_REQUEST_RECEIVED`
- `AXIS_CANCEL_REQUEST_VALIDATED`
- `AXIS_CANCEL_FORWARDED_TO_BACKEND`
- `AXIS_CANCEL_BACKEND_RESULT`
- `AXIS_CANCEL_REJECTED`

`AXIS_CANCEL_BACKEND_RESULT=cancel_packet_sent_no_response_expected` proves
delivery to the backend TCP socket, not that PostgreSQL found a currently
running query or completed cancellation. CancelRequest has no response packet.
If audit intent cannot be recorded before forwarding, AXIS fails closed and
does not contact the backend.

## TLS Transport Decision: Scenario A, Temporary Lab Boundary

> **Status update (2026-08):** The 2026-07-10 inspection below predates the
> mTLS implementation and is superseded by it: client-to-AXIS and
> AXIS-to-backend TLS/mTLS are implemented on the shared write-path transport,
> the CancelRequest path runs over the same mTLS transport, and the negative
> mTLS matrix (18/18, including plaintext and certificate-verification-bypass
> mutations) confirms fail-closed rejection. The inspection text below is
> preserved as the historical record of the 2026-07-10 decision.

Code inspection on 2026-07-10 found no cancel-specific TLS asymmetry because
the current native pgwire transport does not implement TLS on either normal
path:

- client-to-AXIS `SSLRequest` receives `N` in lab mode; with lab mode disabled,
  startup rejects SSLRequest rather than negotiating TLS;
- normal AXIS-to-PostgreSQL sessions use `BackendForwarder`, which opens a raw
  `TcpStream` to `AXIS_PGWIRE_BACKEND_ADDR`;
- `PgwireConfig` has only a backend address and lab-mode flag. It has no TLS
  mode, CA, certificate, server-name, rustls, or native-tls configuration;
- backend CancelRequest forwarding uses the same backend address and raw
  `TcpStream` transport.

Scenario A is therefore retained only as a temporary lab/trusted-network
boundary. It is not justified by claiming that a CancelRequest is insensitive:
the packet contains a backend process ID and secret key, and an on-path observer
who captures both can attempt to cancel that backend session. The short
connection lifetime does not remove that confidentiality risk.

The current threat model excludes hostile or observable links between AXIS and
PostgreSQL. Acceptable test deployments keep this traffic on loopback, an
isolated container network, or an equivalently protected private overlay.
Deploying the current pgwire transport across an untrusted network is not
supported.

Avoiding a TLS handshake also avoids adding latency to a time-sensitive cancel,
but latency is not the primary security justification. The primary reason for
not adding cancel-only TLS is that no normal pgwire TLS connector or
configuration exists to reuse. Implementing a second, cancel-only TLS stack
would create the actual transport asymmetry this decision is intended to avoid.

This decision is temporary, not a permanent production exception. Production
TLS work must introduce one shared backend connector for normal sessions and
CancelRequest connections, define certificate and server-name validation, and
apply the configured TLS requirement consistently. That work must add
`cancel_backend_connection_uses_tls_when_configured` and measure handshake
latency before setting the cancel timeout; the current 75 ms backend timeout is
validated only for cleartext local/private-network tests.

## Current Boundaries

- Client-to-AXIS and AXIS-to-backend pgwire TLS/mTLS are implemented on the
  shared write-path transport; the CancelRequest path runs over the same mTLS
  transport. Plaintext and certificate-verification-bypass variants are
  rejected fail-closed (negative mTLS matrix 18/18). The 2026-07-10 temporary
  lab boundary above is superseded; cleartext traffic is a lab-only
  configuration, not a supported production mode.
- Registry and source rate-limit state are local to one AXIS process; a
  multi-instance deployment requires connection affinity or shared protected
  cancel state.
- The 24-hour TTL limits cancellation for unusually long-lived connections;
  reconnecting obtains a new proxy key.
- Policy authorization beyond possession of the unguessable proxy key is not
  implemented.

## Verification

- `backend_key_mapping_uses_distinct_proxy_key_and_resolves_real_target`
- `backend_key_data_is_substituted_and_valid_cancel_reaches_backend`
- `unknown_cancel_request_is_silently_rejected_and_audited`
- `expired_cancel_request_is_silently_rejected_without_backend_contact`
- `duplicate_cancel_is_suppressed_but_later_cancel_is_allowed`
- `source_rate_limit_rejects_excess_guesses`
- `test_psycopg3_mid_query_cancel_reaches_backend_and_connection_recovers`
- Chaos case `pgwire_concurrent_mid_query_cancel` uses 110 simultaneous
  PostgreSQL sessions, then adds 110 concurrent random key guesses. It verifies
  SQLSTATE `57014`, zero-byte rejection, connection recovery, and process health.
