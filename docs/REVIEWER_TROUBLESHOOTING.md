# Reviewer Troubleshooting

## Requirements

- Docker Desktop must be available.
- Docker Compose must work.
- Docker Hub/package registry access is required.
- Python 3.8+ is required.
- Disconnected/offline environments are not supported in this pilot package.

Corporate network warning:

This demo requires access to Docker Hub and package registries during build. If the reviewer is behind a corporate VPN, SSL inspection proxy, or restricted firewall, the build phase may fail before AXIS itself is running. Such failures should be treated as environment/setup failures, not product enforcement failures.

## Python Command Differences

- Windows: `python`
- macOS/Linux: `python3`

## Platform Check

Windows:

```powershell
python scripts\check_reviewer_platform.py
```

macOS/Linux:

```bash
python3 scripts/check_reviewer_platform.py
```

## Port Conflicts

The pilot expects these local ports:

- frontend: `8088`
- backend: `8000`
- AXIS: `6654`
- Postgres pilot port: `55432`

Stop conflicting local services or edit `demo/docker-compose.pilot.yml` only if you understand the impact on scripts and evidence.

## Docker Commands

Windows:

```powershell
docker compose -f demo/docker-compose.pilot.yml down -v
docker compose -f demo/docker-compose.pilot.yml up -d --build
docker compose -f demo/docker-compose.pilot.yml ps
```

macOS/Linux:

```bash
docker compose -f demo/docker-compose.pilot.yml down -v
docker compose -f demo/docker-compose.pilot.yml up -d --build
docker compose -f demo/docker-compose.pilot.yml ps
```

## Apple Silicon / ARM64

- This pilot should be checked on both `linux/amd64` and `linux/arm64` before broad external distribution.
- Apple Silicon has not been verified by this package unless the final delivery note explicitly says it was tested.
- Docker image architecture mismatch may cause build or runtime failures.
- Such failures should be separated from AXIS policy/enforcement failures.

## Setup Failed

If setup fails and you do not want to debug:

Windows:

```powershell
python scripts\collect_reviewer_diagnostics.py
```

macOS/Linux:

```bash
python3 scripts/collect_reviewer_diagnostics.py
```

Diagnostic output may include Docker logs and platform metadata. The diagnostic collector is designed not to include secrets, `.env` files, SSH keys, tokens, browser data, or unrelated Docker logs. Send the generated ZIP privately.
