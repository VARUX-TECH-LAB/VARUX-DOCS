# AXIS Install Guide

This guide is for reviewers running AXIS locally for technical evaluation. It avoids production deployment assumptions.

## Prerequisites

Windows:

- Git
- Rust toolchain from `rustup`
- Docker Desktop, recommended for the bundled PostgreSQL path
- Node.js LTS
- npm

Linux/macOS:

- Git
- Rust toolchain from `rustup`
- Docker and Docker Compose, recommended for the bundled PostgreSQL path
- Node.js LTS
- npm

## Clone

```cmd
git clone <repository-url>
cd AXIS
```

If you already have the repository locally, run commands from the repository root.

## Backend Setup

Run static and test checks first:

```cmd
cargo fmt --check
cargo check
cargo test
```

For local review with the included PostgreSQL container:

```cmd
docker compose up --build
```

This starts:

- AXIS on `http://localhost:6543`
- PostgreSQL on `localhost:5432`

To run the Rust service directly instead of Docker Compose, PostgreSQL must already be reachable:

```cmd
cargo run
```

## Environment Variables

Backend environment variables are loaded from the process environment and `.env` through `dotenvy`.

| Variable | Default | Purpose |
| --- | --- | --- |
| `LISTEN_ADDR` | `0.0.0.0:6543` | AXIS bind address and port |
| `DATABASE_URL` | Built from DB fields below | PostgreSQL connection string used by `tokio-postgres` |
| `DB_HOST` | `localhost` | PostgreSQL host when `DATABASE_URL` is not set |
| `DB_PORT` | `5432` | PostgreSQL port when `DATABASE_URL` is not set |
| `DB_NAME` | `prod_main` | PostgreSQL database name and default classifier database |
| `DB_USER` | `varux` | PostgreSQL user when `DATABASE_URL` is not set |
| `DB_PASS` | `varux` | PostgreSQL password when `DATABASE_URL` is not set |
| `AXIS_POLICY_DIR` | `./policies` | Directory containing deployable policy files |
| `AXIS_POLICY_MANIFEST` | `./policies/policy_manifest.json` | Authoritative startup manifest for the active policy |
| `AXIS_ENABLE_POLICY_RELOAD` | `false` | Enables internal controlled reload hooks; no HTTP reload endpoint is exposed |
| `POLICY_PATH` | `./policies/prod_main.json` | Deprecated compatibility variable; startup uses `AXIS_POLICY_MANIFEST` |
| `AXIS_POLICY_STORE_PATH` | `./data/policies` | Local policy lifecycle store |
| `AUDIT_WAL_PATH` | `./audit.wal` | Durable audit WAL path |
| `AUDIT_LOG_PATH` | `./audit.log` | JSONL audit projection used by visibility and verification endpoints |
| `AXIS_APPROVAL_DB_PATH` | `./data/approvals.sqlite` | SQLite approval state machine store |
| `OPERATING_MODE` | `enforce` | One of `shadow`, `approval_first`, `enforce`, `emergency_bypass` |
| `AXIS_OPERATOR_TOKEN` | unset | Required token for mutating policy lifecycle endpoints |

These endpoints require `AXIS_OPERATOR_TOKEN` to be configured and a matching `X-AXIS-Operator-Token` header:

- `POST /policy/candidates`
- `POST /policy/activate`
- `POST /policy/rollback`

Control plane variables:

| Variable | Default | Purpose |
| --- | --- | --- |
| `AXIS_CONTROL_PLANE_MODE` | `real` | Use `real` for backend integration or `mock` for explicit UI demo mode |
| `AXIS_BACKEND_URL` | `http://localhost:6543` | Server-side backend target used only by the Next.js proxy |
| `AXIS_PROXY_TIMEOUT_MS` | `8000` | Server-side proxy timeout |
| `NEXT_PUBLIC_REFRESH_INTERVAL_MS` | `5000` | Control plane polling interval |

## Frontend Setup

```cmd
cd control-plane
npm install
npm run build
npm run dev
```

Open:

```text
http://localhost:3000
```

The control plane proxies browser requests through `/api/axis/...` to the server-side `AXIS_BACKEND_URL`. The browser must not call the backend directly. The default backend is:

```text
http://localhost:6543
```

## Smoke Test

Use `curl.exe` on Windows CMD or PowerShell.

```cmd
curl.exe http://localhost:6543/health
curl.exe http://localhost:6543/runtime/stats
curl.exe "http://localhost:6543/logs?limit=10"
curl.exe http://localhost:6543/evidence/verify
```

Run a safe read:

```cmd
curl.exe -X POST http://localhost:6543/query -H "Content-Type: application/json" -d "{\"actor\":\"reviewer\",\"app\":\"demo\",\"tenant\":\"acme\",\"role\":\"developer\",\"host\":\"localhost\",\"env\":\"prod\",\"sql\":\"SELECT 1 AS axis_demo\"}"
```

Run a risky write that should be blocked by the default policy:

```cmd
curl.exe -X POST http://localhost:6543/query -H "Content-Type: application/json" -d "{\"actor\":\"reviewer\",\"app\":\"demo\",\"tenant\":\"acme\",\"role\":\"developer\",\"host\":\"localhost\",\"env\":\"prod\",\"sql\":\"DELETE FROM users\"}"
```

Inspect recent evidence:

```cmd
curl.exe "http://localhost:6543/audit?limit=10"
curl.exe http://localhost:6543/evidence/verify
```

## Troubleshooting

`npm` is not recognized:

Install Node.js LTS and reopen the terminal so `node` and `npm` are on `PATH`.

`cargo` or `rustup` is not recognized:

Install Rust from `rustup`, then reopen the terminal.

Port already in use:

AXIS defaults to `6543`, the control plane defaults to `3000`, and Docker PostgreSQL defaults to `5432`. Stop the conflicting service or change `LISTEN_ADDR` / frontend port as needed.

Docker is not running:

Start Docker Desktop or the Docker daemon before `docker compose up --build`.

Backend is not reachable from the control plane:

Check server-side `AXIS_BACKEND_URL`, then confirm `curl.exe http://localhost:6543/health` works. From the Control Plane proxy, confirm `curl.exe http://localhost:3000/api/axis/health` and `curl.exe "http://localhost:3000/api/axis/logs?limit=10"` work.

Policy file not found:

Check `POLICY_PATH` and confirm `policies/prod_main.json` exists. If using Docker Compose, the compose file mounts `./policies` into `/app/policies`.

Policy lifecycle store cannot be written:

Check `AXIS_POLICY_STORE_PATH`. The default `./data/policies` must be writable by the AXIS process.

Audit log malformed or evidence verification is invalid:

Run `GET /audit?limit=10` and inspect backend logs. Legacy or manually edited records can make verification report invalid; AXIS should not silently repair evidence.

Operator token missing:

If `AXIS_OPERATOR_TOKEN` is set, include `X-AXIS-Operator-Token` on mutating policy lifecycle requests. Read-only policy endpoints do not require the token.

Database connection fails on `cargo run`:

Use `docker compose up --build` for the bundled local database, or set `DATABASE_URL` / DB fields for your PostgreSQL instance.
