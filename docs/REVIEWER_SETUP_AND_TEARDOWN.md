# Reviewer Setup And Teardown

## Start Stack

```bash
docker compose -f demo/docker-compose.pilot.yml up -d --build
```

## Stop Only

```bash
docker compose -f demo/docker-compose.pilot.yml down
```

## Clean Docker Volumes

```bash
docker compose -f demo/docker-compose.pilot.yml down -v
```

## Safe Cleanup

Windows:

```powershell
python scripts\clean_reviewer_environment.py
```

macOS/Linux:

```bash
python3 scripts/clean_reviewer_environment.py
```

## Dry Run

Windows:

```powershell
python scripts\clean_reviewer_environment.py --dry-run
```

macOS/Linux:

```bash
python3 scripts/clean_reviewer_environment.py --dry-run
```

## Non-Interactive Cleanup

Windows:

```powershell
python scripts\clean_reviewer_environment.py --yes
```

macOS/Linux:

```bash
python3 scripts/clean_reviewer_environment.py --yes
```

## What Is Removed

- Pilot Docker containers created by `demo/docker-compose.pilot.yml`.
- Pilot Docker volumes when `down -v` runs.
- Package-specific generated host artifacts such as `diagnostics/`, `demo-evidence-output/`, and local pilot runtime log folders when present and confirmed.

## What Is Intentionally Not Removed

- The source package itself.
- `dist/AXIS-External-Reviewer-Package-v1/`.
- Pre-generated evidence.
- Existing source files.
- Unrelated Docker images, containers, volumes, or networks.
- Global caches.
- Home directory files.
- Unrelated project folders.

The cleanup script is intentionally conservative. It removes AXIS reviewer-package artifacts only and does not clean unrelated Docker resources.

## Manual Inspection

To inspect remaining Docker resources:

```bash
docker compose -f demo/docker-compose.pilot.yml ps
docker volume ls
docker image ls
```

To inspect host artifacts, check:

- `diagnostics/`
- `demo-evidence-output/`
- `dist/AXIS-External-Reviewer-Package-v1/`
