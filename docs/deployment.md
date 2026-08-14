# AXIS Deployment

This document covers local reviewer deployment for AXIS with Docker Compose. It is not a managed production runbook.

## Prerequisites

- Docker Desktop or Docker Engine with Compose v2.
- Rust toolchain for local backend validation.
- Python 3.10+ for smoke, verifier, regression, restart, chaos, and stress scripts.
- Node.js 20+ for the Control Plane.

Run commands from the repository root unless noted.

## Local Demo Startup

The default Compose mode starts PostgreSQL and the AXIS backend on `http://localhost:6543`. Port `6544` also maps to the same backend for validation scripts that need to avoid a stale local listener.

```powershell
docker compose down
docker compose up --build -d
docker compose ps
```

Check the runtime:

```powershell
curl.exe http://localhost:6543/health
curl.exe "http://localhost:6543/runtime/metrics"
curl.exe "http://localhost:6543/audit/verify"
```

The default profile is `local`. It uses a local-only placeholder operator token and unsigned Evidence Bundle V1 export.

## Demo Profile

The demo profile is the default local Compose deployment with:

- PostgreSQL service `postgres`.
- AXIS backend service `dbguard`.
- Policy mounted read-only from `./policies`.
- Audit WAL, JSONL projection, approvals, and policy store held in named volume `dbguard_data`.
- Local export auth disabled by default with `AXIS_AUDIT_EXPORT_REQUIRES_OPERATOR=false`.

Run smoke and stress checks:

```powershell
python scripts/axis_audit_api_smoke.py --base http://localhost:6543
python scripts/axis_runtime_smoke.py --base http://localhost:6543
python scripts/axis_runtime_stress.py --base http://localhost:6543 --concurrency 25 --requests 500 --include-export --include-rate-limit
```

## Production-Like Profile

Production-like mode uses the `dbguard-production-like` Compose service and publishes `http://localhost:6545`. It sets `AXIS_RUNTIME_PROFILE=production`, requires a strong `AXIS_OPERATOR_TOKEN`, and defaults audit export auth to enabled.

Create a private env file from `.env.production.example` and replace placeholders:

```powershell
Copy-Item .env.production.example .env.production.local
notepad .env.production.local
docker compose --env-file .env.production.local --profile production-like up --build -d postgres dbguard-production-like
docker compose --profile production-like ps
```

Validate:

```powershell
curl.exe http://localhost:6545/health
curl.exe "http://localhost:6545/runtime/metrics"
curl.exe -H "X-AXIS-Operator-Token: <your-token>" "http://localhost:6545/audit/export?limit=5" -o axis-evidence-bundle-v1.json
python scripts/axis_evidence_bundle_verify.py --bundle axis-evidence-bundle-v1.json
```

Do not commit `.env.production.local`.

## Environment Variables

Required or commonly reviewed backend variables:

- `AXIS_RUNTIME_PROFILE`: `local` or `production`.
- `AXIS_OPERATOR_TOKEN`: required and strong when profile is `production`.
- `AXIS_REQUEST_TIMEOUT_MS`, `AXIS_DB_QUERY_TIMEOUT_MS`, `AXIS_DB_CONNECT_TIMEOUT_MS`.
- `AXIS_DB_POOL_MAX_CONNECTIONS`, `AXIS_DB_POOL_ACQUIRE_TIMEOUT_MS`.
- `AXIS_RATE_LIMIT_ENABLED`, `AXIS_RATE_LIMIT_REQUESTS_PER_MINUTE`, `AXIS_RATE_LIMIT_BURST`.
- `AXIS_MAX_BODY_BYTES`, `AXIS_MAX_SQL_BYTES`.
- `AXIS_EVIDENCE_SIGNING_ENABLED`, `AXIS_EVIDENCE_SIGNING_KEY_ID`, `AXIS_EVIDENCE_SIGNING_PRIVATE_KEY_B64`, `AXIS_EVIDENCE_SIGNING_PUBLIC_KEY_B64`.
- `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASS`, `DB_NAME`, or `DATABASE_URL`.
- `AXIS_POLICY_DIR`, `AXIS_POLICY_MANIFEST`, `POLICY_PATH`.
- `AUDIT_WAL_PATH`, `AUDIT_LOG_PATH`, `AUDIT_INDEX_PATH`, `AXIS_APPROVAL_DB_PATH`, `AXIS_POLICY_STORE_PATH`.

`.env.example` is local-only. `.env.production.example` is a placeholder template and contains no real secrets.

## Runtime Validation Commands

```powershell
cargo fmt
cargo check
cargo test

docker compose down
docker compose up --build -d
docker compose ps

curl.exe http://localhost:6543/health
curl.exe "http://localhost:6543/runtime/metrics"
curl.exe "http://localhost:6543/audit/events?limit=5"
curl.exe "http://localhost:6543/audit/verify"
curl.exe "http://localhost:6543/audit/export?limit=5" -o axis-evidence-bundle-v1.json

python scripts/axis_evidence_bundle_verify.py --bundle axis-evidence-bundle-v1.json
python scripts/axis_audit_api_smoke.py --base http://localhost:6543
python scripts/axis_runtime_smoke.py --base http://localhost:6543
python scripts/axis_runtime_stress.py --base http://localhost:6543 --concurrency 25 --requests 500 --include-export --include-rate-limit
python axis_regression.py --base http://localhost:6543
python axis_audit_restart_test.py --base http://localhost:6543
python axis_chaos_test.py --base http://localhost:6543
```

Control Plane:

```powershell
cd control-plane
npm run typecheck
npm run lint
npm run build
npm run smoke:real
npx playwright test e2e/real-mode.spec.ts
```

## Evidence Export

Export a small bundle:

```powershell
curl.exe "http://localhost:6543/audit/export?limit=10" -o axis-evidence-bundle-v1.json
python scripts/axis_evidence_bundle_verify.py --bundle axis-evidence-bundle-v1.json
```

If signing is enabled, pass the public key and require a signature:

```powershell
python scripts/axis_evidence_bundle_verify.py --bundle axis-evidence-bundle-v1.json --public-key-b64 <public-key> --require-signature
```

## Data Volume Behavior

`dbguard_data` stores `/app/data`, including:

- `audit.wal`: canonical WAL-backed audit evidence.
- `audit.log`: JSONL projection for operator convenience.
- `approvals.sqlite`: SQLite approval state machine store.
- policy store files.

`pgdata` stores PostgreSQL data. `docker compose down` stops services but keeps volumes. Reset local state with:

```powershell
docker compose down -v
```

This deletes local Docker volumes. Do not run it against an environment whose evidence you need to preserve.

## Policy Manifest Behavior

The active policy is loaded from `AXIS_POLICY_MANIFEST`, which points at `policies/policy_manifest.json` in Compose. The manifest checksum and policy version must match the active policy file. Startup fails fast on missing policy files, malformed policy JSON, checksum mismatch, or policy version mismatch.

## Operator Token Behavior

Policy mutation and approval resolution require operator auth. Audit export requires operator auth when `AXIS_AUDIT_EXPORT_REQUIRES_OPERATOR=true`, which is the production-profile default. Tokens are supplied server-side and are not exposed to browser code.

## Signing Behavior

Evidence signing is disabled by default. When enabled, AXIS expects base64 Ed25519 key material through server-only environment variables. Private keys must not be committed, copied into images, or exposed through frontend env variables.

## Known Limitations

- This Compose setup is local reviewer packaging, not a high-availability deployment.
- WAL-backed audit reads and exports use Audit Derived Index V1 for candidate selection when the index is ready. Event bodies, verification, and Evidence Bundle V1 integrity remain WAL-backed.
- Evidence Bundle V1 proves included AXIS evidence integrity, not external business truth.
- Production-like mode demonstrates fail-fast config and operator auth behavior but does not replace secret management, TLS termination, backups, or monitoring.
