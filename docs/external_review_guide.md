# AXIS External Review Guide

## 1. AXIS Overview

AXIS is a database access guard that validates requests, classifies SQL, applies deterministic policy, records WAL-backed audit evidence, and exposes review APIs for audit, runtime metrics, policy status, approvals, and Evidence Bundle V1 export.

## 2. What AXIS Protects

- Production database write paths governed by policy.
- Approval-gated operations.
- Audit evidence continuity through a hash chain.
- Evidence export integrity through bundle hashing and optional Ed25519 signing.
- Runtime behavior under malformed requests, timeouts, rate limits, and pool pressure.

## 3. What AXIS Does Not Protect Yet

- Network perimeter or identity provider integration.
- Full PostgreSQL protocol proxying.
- High-availability storage or backup orchestration.
- Indexed audit search at large scale.
- External business truth outside AXIS evidence.

## 4. Core Runtime Flow

HTTP request -> request validation -> SQL classification -> policy decision -> WAL audit evidence -> DB execution if allowed -> result evidence -> response -> optional audit verification or Evidence Bundle V1 export.

Protected writes require durable decision evidence before DB execution. AXIS does not silently repair or skip malformed audit records.

## 5. Policy Decision Flow

AXIS loads the active policy through `AXIS_POLICY_MANIFEST`, validates manifest integrity, runs startup dry-run checks, classifies the request, evaluates defaults and rules, and returns `ALLOW`, `BLOCK`, or `REQUIRE_APPROVAL`.

## 6. Approval Workflow

Approval-required decisions create immutable approval records. Resolution requires operator auth. Resolution produces audit evidence and should not allow double execution during concurrent resolve races.

## 7. Audit And Evidence Model

The WAL is canonical. The JSONL audit projection is operator convenience. Verification recomputes continuity and reports `verified`, `tampered`, `unverifiable`, or `error` honestly.

## 8. Evidence Bundle V1 Verification

Export:

```powershell
curl.exe "http://localhost:6543/audit/export?limit=10" -o axis-evidence-bundle-v1.json
python scripts/axis_evidence_bundle_verify.py --bundle axis-evidence-bundle-v1.json
```

Signed bundles can be verified with:

```powershell
python scripts/axis_evidence_bundle_verify.py --bundle axis-evidence-bundle-v1.json --public-key-b64 <public-key> --require-signature
```

## 9. Runtime Guardrails

Review:

- Request timeout.
- DB query and connect timeouts.
- DB pool acquisition timeout.
- Max request body and SQL sizes.
- Rate limiting.
- Startup fail-fast config validation.
- Structured errors without raw SQL, secrets, private keys, or filesystem paths.

## 10. Threat Model Summary

Primary threats are unsafe production writes, approval bypass, audit tampering, malformed request abuse, evidence export tampering, secret leakage, and operational failure under pressure. Current controls are deterministic policy, immutable approvals, WAL-backed evidence, hash-chain verification, redacted bundle export, optional signing, operator auth, timeouts, pool bounds, rate limiting, and Docker-backed validation.

## 11. Security Assumptions

- The backend environment is trusted to hold server-only secrets.
- PostgreSQL credentials and operator tokens are supplied by deployment tooling.
- Reviewers preserve audit volumes when evidence continuity matters.
- Browser code uses the Control Plane proxy and does not receive backend URLs or operator tokens.

## 12. Known Limitations

- Local Compose is not production HA.
- No external identity provider integration.
- Audit explorer reads use a derived index for candidate selection when available. WAL remains canonical, index rebuild is synchronous today, and `/audit/verify` remains a full WAL verification scan.
- Evidence bundles do not include raw WAL records.
- Unsigned bundles are hash-protected but not attributable to a signing key.

## 13. How To Run The System

```powershell
docker compose down
docker compose up --build -d
docker compose ps
curl.exe http://localhost:6543/health
```

Production-like:

```powershell
Copy-Item .env.production.example .env.production.local
notepad .env.production.local
docker compose --env-file .env.production.local --profile production-like up --build -d postgres dbguard-production-like
curl.exe http://localhost:6545/health
```

## 14. How To Run Tests

```powershell
cargo fmt
cargo check
cargo test
python scripts/axis_audit_api_smoke.py --base http://localhost:6543
python scripts/axis_runtime_smoke.py --base http://localhost:6543
python axis_regression.py --base http://localhost:6543
python axis_audit_restart_test.py --base http://localhost:6543
python axis_chaos_test.py --base http://localhost:6543
```

Frontend:

```powershell
cd control-plane
npm run typecheck
npm run lint
npm run build
npm run smoke:real
npx playwright test e2e/real-mode.spec.ts
```

## 15. How To Run Stress Validation

```powershell
python scripts/axis_runtime_stress.py --base http://localhost:6543 --concurrency 25 --requests 500 --include-export --include-rate-limit
```

Expected result:

```text
AXIS Runtime Stress Validation: PASS
```

A failure means the runtime produced an unsafe decision, unstructured error, unverifiable audit chain, broken evidence bundle, or leaked forbidden text.

## 16. What Reviewers Should Inspect

- `src/gate/listener.rs`
- `src/gate/enforcer.rs`
- `src/audit/logger.rs`
- `src/audit/reader.rs`
- `src/audit/verifier.rs`
- `src/audit/evidence_bundle.rs`
- `src/config.rs`
- `src/db/postgres.rs`
- `policies/policy_manifest.json`
- `policies/prod_main.json`
- `control-plane/src/app/api/axis/[...path]/route.ts`
- `Dockerfile`
- `docker-compose.yml`
- `.env.example`
- `.env.production.example`

## 17. Expected Review Questions

- What happens if audit WAL append fails before a protected write?
- Can a malformed audit record be skipped or repaired silently?
- Can an approval be resolved twice under concurrency?
- Are dangerous writes ever allowed under DB or audit failure?
- Does the browser receive `AXIS_OPERATOR_TOKEN`, signing keys, or backend filesystem paths?
- Are Evidence Bundle V1 hashes and signatures verifiable offline?
- Does startup fail on missing or mismatched policy manifests?
- Are stress and chaos results reproducible from documented commands?
