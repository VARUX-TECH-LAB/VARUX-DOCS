# Reviewer Video Demo Script

VIDEO_LINK_PLACEHOLDER: add private demo link here before external delivery.

Purpose: provide a precise 3-minute demo script for Loom, YouTube Unlisted, or private video recording.

Tone: practical, not salesy. No founder story. No hype. No native wire claim.

## Script

0:00 - 0:20

Say: "AXIS is a deterministic control layer for protected PostgreSQL write paths."

Show the package title and `demo/REVIEWER_START_HERE.md`.

0:20 - 0:45

Show reviewer package / Quickstart. Point to the four paths: 3-minute watch, no-run review, local run, setup failed.

0:45 - 1:20

Show clean pilot stack health:

```bash
docker compose -f demo/docker-compose.pilot.yml ps
python scripts\check_reviewer_platform.py
```

Open or show health evidence under `demo/evidence/pilot-v1/health/`.

1:20 - 1:50

Trigger safe write through ORM-backed business app:

```bash
python scripts\run_pilot_demo.py
```

Show "ORM customer insert routed through AXIS" and the returned audit event id.

1:50 - 2:20

Trigger risky operation and show `approval_required`. Show approval id only from the live response; do not invent one.

2:20 - 2:40

Show approval retry success after explicit approval resolution. Explain that the original DB transaction is not held open while waiting for approval.

2:40 - 3:00

Trigger blocked destructive operation and show audit evidence location:

- `demo/evidence/pilot-v1/responses/blocked-operation-response.json`
- `demo/evidence/pilot-v1/audit/audit-sample-events.json`

Close by asking for a 15-minute technical teardown.
