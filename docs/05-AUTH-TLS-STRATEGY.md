---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---


# Auth and TLS Strategy

## Purpose

This document defines authentication and TLS strategy for AXIS Native PG mode.

## PoC Mode

PoC mode may use cleartext local TCP and startup/auth pass-through.

This is acceptable only for lab work.

## Critical PoC Limitation

In auth pass-through mode, AXIS does not verify the client's identity. It relays the backend authentication exchange. The observed database user is `unauthenticated_claimed_db_user` from AXIS's perspective.

This must be stated in audit and documentation.

## TLS Modes

### TLS Pass-Through

AXIS forwards encrypted bytes and cannot inspect SQL.

Status: not useful for policy enforcement.

### TLS Termination at AXIS

Client establishes TLS to AXIS. AXIS decrypts traffic, enforces policy, then opens a separate TLS/plain connection to backend depending on deployment.

Status: required direction for real enforcement.

### mTLS Termination

Client authenticates to AXIS with certificates. AXIS maps cert identity to actor/service identity and opens backend connection with controlled credentials.

Status: preferred enterprise target.

## SCRAM

SCRAM pass-through is complex and weak for AXIS identity assurance.

If AXIS simply relays SCRAM, backend authenticates the client but AXIS still cannot claim it independently verified identity unless it validates the exchange or trusts backend response as identity proof.

Enterprise target should evaluate:

- AXIS-managed auth.
- mTLS.
- OIDC/LDAP integration.
- Service account backend access.
- Per-actor backend auth only where operationally feasible.

## Backend Service Account

Service account backend access is acceptable for PoC and may be acceptable in enterprise if identity attribution is preserved via AXIS audit and backend correlation.

Risks:

- PostgreSQL native logs show axis_service.
- Database-level RBAC may be weakened.
- Compromise of AXIS service account is high impact.

Mitigations:

- Least-privilege service account.
- Per-environment credentials.
- Correlation ID injection via controlled session metadata.
- Strict block of client attempts to modify AXIS-managed metadata.
- Optional RLS/custom GUC strategy.

## Certificate Rotation

Enterprise mode must support:

- Reloading certs without dropping existing sessions where possible.
- New sessions using new certs.
- Expiry monitoring.
- mTLS CA bundle rotation.
- Operator-visible certificate health.

This is not optional for production. Certificates expire because apparently time itself enjoys breaking infrastructure.

## Current Known Weaknesses

- PoC pass-through auth is not identity verification.
- SCRAM strategy is unresolved.
- mTLS target requires operational tooling.
- Service account model requires strong attribution and monitoring.

## Acceptance Criteria

Auth/TLS strategy is acceptable only when AXIS can clearly say which identity it verified, which identity it merely observed, and how that distinction appears in audit evidence.
