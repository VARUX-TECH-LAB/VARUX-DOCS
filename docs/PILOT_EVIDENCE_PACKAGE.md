# AXIS Pilot Evidence Package

## Directory map

```text
evidence/
  pilot-v1/
    README.md
    clean-run/
      commands.md
      smoke-test-output.txt
      demo-run-output.txt
      compose-ps-output.txt
    health/
      axis-health.json
      backend-health.json
      frontend-health.txt
    responses/
      safe-read-response.json
      safe-write-response.json
      approval-required-response.json
      approval-retry-success-response.json
      approval-rejected-response.json
      blocked-operation-response.json
      transaction-safe-response.json
      transaction-approval-rollback-response.json
      transaction-blocked-rollback-response.json
    audit/
      audit-sample-events.json
      audit-verification-output.json
      audit-hash-chain-notes.md
    screenshots/
      README.md
    limitations/
      current-limitations.md
      native-wire-gap.md
      sql-alchemy-integration-notes.md
      approval-retry-model.md
      transaction-model.md
```

## How evidence is captured

`scripts/capture_pilot_evidence.py` calls the running pilot stack over HTTP. It does not generate fake policy decisions, fake approval ids, fake audit ids, or placeholder screenshots.

The script captures:

- AXIS health from `GET http://localhost:6654/health`.
- Backend health from `GET http://localhost:8000/health`.
- Frontend reachability from `GET http://localhost:8088`.
- Clean-run command outputs for smoke tests, demo run, and compose service status.
- Business responses for safe read, safe write, approval, rejection, blocked operation, and transaction workflows.
- AXIS audit events from `GET /audit/events?limit=100`.
- AXIS audit hash-chain verification from `GET /audit/verify` when available.

If capture stops before completion, it leaves `demo/evidence/pilot-v1/PARTIAL_CAPTURE.txt`. The verifier treats that as a failure.

## How to regenerate evidence

From the repository root:

```powershell
docker compose -f demo/docker-compose.pilot.yml down -v
docker compose -f demo/docker-compose.pilot.yml up -d --build
python scripts\pilot_smoke_tests.py
python scripts\run_pilot_demo.py
python scripts\capture_pilot_evidence.py
python scripts\verify_pilot_evidence.py
```

The capture script also archives smoke-test and demo output by invoking those same Python scripts during capture. This ensures the evidence directory contains reviewable command output even when a reviewer runs the commands interactively.

## How to verify evidence

Run:

```powershell
python scripts\verify_pilot_evidence.py
```

Expected success output:

```text
PILOT_EVIDENCE_VERIFICATION: PASS
```

On failure, the verifier prints each missing or invalid evidence item and then:

```text
PILOT_EVIDENCE_VERIFICATION: FAIL
```

## What each response file proves

- `safe-read-response.json`: the sample app can read business data directly from PostgreSQL and labels that path as `direct_postgres`.
- `safe-write-response.json`: a normal SQLAlchemy ORM insert is routed through AXIS, allowed by policy, executed, and audited.
- `approval-required-response.json`: a risky role update returns structured `approval_required` state with an AXIS `approval_id`.
- `approval-retry-success-response.json`: an approved operation succeeds only after the original request is explicitly retried with `approval_id`.
- `approval-rejected-response.json`: a rejected approval blocks the retry and returns structured `approval_rejected` state.
- `blocked-operation-response.json`: an unscoped destructive delete is blocked by AXIS policy and not executed.
- `transaction-safe-response.json`: a multi-write business workflow can route multiple safe writes through AXIS and complete.
- `transaction-approval-rollback-response.json`: a workflow that hits an approval-required operation rolls back local pending state; `marker_persisted` and the state check must both prove no partial marker.
- `transaction-blocked-rollback-response.json`: a workflow that hits a blocked destructive operation rolls back local pending state.

## How audit evidence should be inspected

Start with `audit-verification-output.json`. For the current pilot stack, `/audit/verify` should report `status: verified` and a positive `events_checked` count.

Then inspect `audit-sample-events.json`. Reviewers should look for:

- `event_id`
- `decision`
- `reason` or `reason_code`
- `risk`
- `matched_rule`
- `policy.policy_id`, `policy.policy_version`, and `policy.policy_sha256`
- `sql_fingerprint`
- `approval_id` for approval-related events
- `event_hash`
- `previous_hash`

`audit-hash-chain-notes.md` explains what AXIS verified automatically and what remains manual or future work.

## How to interpret PASS/FAIL results

`PASS` means the evidence directory is complete and internally consistent for the current pilot claim.

`FAIL` means a reviewer should not treat the package as complete. The printed failure lines identify exactly which file, field, command output, policy state, rollback state, or audit item needs attention.
