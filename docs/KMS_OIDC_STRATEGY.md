# AXIS KMS and OIDC Strategy

This document describes the intended production direction. It does not claim that AXIS currently implements full KMS, Vault, HSM, OIDC, or JWKS network integration.

## Current State

AXIS supports local/demo JWT HS256 authentication for authenticated context validation. HS256 is useful for deterministic local and reviewer checks because it proves that AXIS derives actor, tenant, role, app, host, actor type, and environment from a verified bearer token instead of trusting request JSON fields.

HS256 is not full enterprise IAM.

Evidence bundle signing supports local Ed25519 key material and now reports signature metadata honestly:

- `signature_status`
- `signature_algorithm`
- `kid` / `key_id`
- `signed_at` when signed

Unsigned exports are marked disabled/unsigned and are not presented as signed evidence.

## Production Identity Direction

Production identity should use one or more trusted identity mechanisms:

- OIDC/JWKS validation against an enterprise identity provider
- mTLS with certificate identity mapping
- cloud IAM or workload identity
- service identity from the deployment platform
- short-lived credentials issued by an approved control plane

The future OIDC/JWKS adapter should validate issuer, audience, expiry, signature, key id, and claims mapping without trusting body-supplied identity fields.

## Production Signing Key Direction

Production evidence signing keys should use managed key infrastructure where possible:

- cloud KMS
- HSM
- managed secret store with audited access
- Vault-backed signing adapter
- hardware-backed or service-backed signing where available

Private signing material should not be committed, copied into container images, returned through APIs, or written into reports.

## Adapter Direction

Future adapters should keep the current honest metadata model:

- report `signature_status: disabled` when signing is not configured
- report `signature_status: signed` only after a real signature is produced
- report `signature_status: ephemeral_demo` only for explicitly ephemeral demo signing
- include `kid` for key identity and rotation review
- fail safely when signing is required but key material or remote signing is unavailable

This hardening work prepares the metadata and redaction boundaries. It does not implement automatic rotation, external key store connectivity, or network identity-provider validation.
