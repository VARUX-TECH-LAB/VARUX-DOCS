# AXIS Evidence Bundle V1

An AXIS Evidence Bundle is a portable JSON export of durable audit evidence from
`GET /audit/export`. It is intended for customer, auditor, and security review
handoff without requiring a running AXIS backend.

## Schema Version 1.0

Evidence Bundle V1 uses:

- `bundle_type`: `axis.evidence_bundle.v1`
- `schema.name`: `axis.evidence_bundle`
- `schema.version`: `1.0`
- `integrity.canonicalization`: `axis.canonical_json.v1`

Schema version `1.0` means the top-level object layout, integrity fields,
redaction contract, and verifier behavior are stable for V1 consumers. Future
breaking changes must use a new schema version or bundle type.

## Bundle Contents

The bundle includes:

- Bundle metadata: `bundle_id`, `generated_at`, generator identity, filters, and
  summary counts.
- Verification report from the AXIS audit verifier at export time.
- Active policy evidence: policy id, policy version, policy SHA-256, manifest
  loaded state, and policy integrity state.
- Exported audit events in a share-safe summary format.
- Integrity metadata: canonical payload hash, event-hash aggregate hash, optional
  Ed25519 signature metadata, and signing timestamp.
- Redaction metadata that states secrets, filesystem paths, and raw fields were
  excluded or redacted.

The bundle does not include raw audit records, signing private keys, operator
tokens, backend-only environment variables, or backend filesystem paths.

## What It Does Not Prove

An Evidence Bundle proves that the included AXIS evidence was exported in a
specific V1 structure and that the exported payload has not changed since the
recorded hash or signature was produced.

It does not prove external business truth. For example, it does not prove that a
third-party ticket, customer request, or database state outside AXIS was correct.
It also does not prove evidence that was not included in the export.

## Canonicalization

`axis.canonical_json.v1` is deterministic JSON used for hashing and signing:

- Object keys are ordered lexicographically.
- Arrays preserve their exported order.
- Null values are represented deterministically.
- Timestamps are RFC3339 strings.
- Whitespace is not part of the canonical payload.
- `integrity.payload_sha256` is excluded from the canonical payload because it is
  self-referential.
- `integrity.signature` is excluded from the canonical payload because the
  signature signs the payload, not itself.
- Other integrity metadata, including `signature_algorithm`,
  `signature_status`, `kid`, `signature_key_id`, and `signed_at`, is included
  in the payload hash and signature.

`event_hashes_sha256` is the SHA-256 of the canonical JSON array of included
non-empty `event_hash` values, in exported event order. If no event hashes are
included, it is `null`.

## Signing

Signing is optional and server-side only. AXIS supports Ed25519 signing when
configured with:

- `AXIS_EVIDENCE_SIGNING_ENABLED=true`
- `AXIS_EVIDENCE_SIGNING_KEY_ID=<key id>`
- `AXIS_EVIDENCE_SIGNING_PRIVATE_KEY_B64=<base64 raw private key>`
- `AXIS_EVIDENCE_SIGNING_PUBLIC_KEY_B64=<base64 raw public key>`

When signing is disabled, exports still work and are marked:

- `signature_algorithm`: `none`
- `signature_status`: `disabled`
- `kid`: `null`
- `signature_key_id`: `null`
- `signature`: `null`
- `signed_at`: `null`

Unsigned bundles are honest portable evidence, but they are not cryptographically
attributable to a signing key. Reviewers should treat them as hash-protected
exports and should obtain them through a trusted channel.

When signing is enabled and key material is valid, AXIS signs the canonical
unsigned payload with Ed25519 and records `signature_status: signed`, the key id
in `kid` and `signature_key_id`, and the signing timestamp. If signing is
enabled but key material is invalid, export fails safely with a
structured error and does not leak key material.

## Verification Scope

`verification.verification_scope` describes what AXIS checked at export time:

- `chain_continuity`: previous hash continuity only.
- `chain_continuity_and_event_hash`: previous hash continuity and event hash
  recomputation were checked.

The offline verifier checks the exported bundle structure, payload hash,
event-hash aggregate, and signature when applicable. It does not recompute AXIS
WAL event hashes from raw WAL records because raw records are not included.

## Redaction

Evidence Bundle V1 is designed for sharing. Exported events omit raw audit
records and raw SQL fields. Secret-like fields, private keys, operator tokens,
credentials, and backend filesystem paths are not included. The `redaction`
object records the redaction posture:

- `secrets_redacted: true`
- `filesystem_paths_redacted: true`
- `raw_fields_included: false`

## Verify a Bundle

Verify an unsigned or signed bundle structure and hashes:

```bash
python scripts/axis_evidence_bundle_verify.py --bundle axis-evidence-bundle-v1.json
```

Verify a signed bundle and require a valid signature:

```bash
python scripts/axis_evidence_bundle_verify.py --bundle axis-evidence-bundle-v1.json --public-key-b64 <key> --require-signature
```

Expected verifier outcomes:

- `AXIS Evidence Bundle Verification: PASS`
- `AXIS Evidence Bundle Verification: PASS_WITH_WARNINGS` with
  `reason: bundle_unsigned`
- `AXIS Evidence Bundle Verification: FAIL` with a reason such as
  `payload_sha256_mismatch`

The verifier returns a non-zero exit code on failure.

## Known Limitations

WAL reads are still linear. Large audit logs can make export and verification
latency proportional to the WAL size.

An Evidence Bundle proves included AXIS evidence, not external business truth.
