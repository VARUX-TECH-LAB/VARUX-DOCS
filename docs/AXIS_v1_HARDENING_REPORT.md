# AXIS v1 Hardening Report

Status date: 2026-05-10

AXIS v1 is not a production-ready enterprise product. It is a local hardened v1 security core for deterministic SQL gate enforcement, policy regression, approval flow validation, audit evidence, and failure-mode testing.

## Current Verdict

Hardening phase: ACTIVE+

| Area | Verdict |
| --- | --- |
| AXIS v1 runtime | PASS |
| Parser bypass corpus baseline | PASS |
| Policy regression baseline | PASS |
| Approval regression baseline | PASS |
| Audit-chain restart continuity | PASS |
| Corrupt audit fail-fast | PASS |
| Chaos/failure-mode baseline | PASS |

## What Was Proven

The current v1 baseline proves the following behavior in the local hardened core:

- Read, write, block, and approval paths are working.
- Risky SQL does not receive unsafe `200 ALLOW`.
- Parser bypass corpus baseline passed.
- Policy regression baseline passed: 31/31 total regression cases.
- Approval regression baseline passed.
- Stress baseline passed with 1000 requests, 50 concurrency, and unexpected result count 0.
- Audit chain continuity across restart is preserved: the first post-restart `previous_hash` continues from the previous event `event_hash`.
- Corrupt audit startup fail-fast behavior is verified.
- Chaos suite passed: 10/10 scenarios.
- The tested chaos scenarios include PostgreSQL down, audit unwritable, invalid policy, corrupt approval store, malformed JSON, huge payload, concurrent approval race, restart during traffic, DB pool pressure, and corrupt audit startup.

## Security Boundary

AXIS v1 sits in front of the database write path and evaluates incoming SQL before execution. It classifies SQL, applies the configured policy, writes audit evidence, and returns one of these outcomes:

- `ALLOW`: query is executed and audited.
- `BLOCK`: query is not executed and the block is audited.
- `REQUIRE_APPROVAL`: query is queued for approval and not executed until a valid approval resolution occurs.

The current policy file is `policies/prod_main.json`. The default operating mode in local configuration is `enforce`.

The v1 guarantee is local and process-bound. It does not claim distributed, multi-instance, production-grade audit finality or enterprise access control.

## Evidence Sources

Primary local evidence sources:

- `axis_regression.py`
- `axis_gate_stress.py`
- `axis_audit_restart_test.py`
- `axis_chaos_test.py`
- `tests/policy_cases.json`
- `tests/parser_bypass_cases.json`
- `tests/approval_cases.json`
- `policies/prod_main.json`
- `audit.log`
- `approvals.jsonl`

Main validation commands are documented in `docs/technical/AXIS_TEST_COMMANDS.md`.

## Interpreted Result

The local v1 hardening work is sufficient to call AXIS a hardened security core for local pre-execution SQL gate validation. The evidence supports deterministic policy behavior, controlled rejection for malformed input, approval-store behavior under race pressure, audit-chain restart continuity, and fail-safe behavior under the tested chaos cases.

This does not make AXIS production-ready. It means the v1 core has a meaningful evidence baseline and known failure behavior.

## Remaining Risks

The following risks remain open and should be treated as explicit non-goals or future hardening work:

- No auth, TLS, or RBAC.
- No metrics or observability pipeline.
- No log rotation.
- Recovery runbook is initial-level only.
- No full historical audit-chain verification.
- No multi-instance distributed audit chain.
- No long-running soak test.
- No production deployment hardening.

## README Current Status Draft

The following short section can be added to `README.md`:

```markdown
## Current Status

AXIS v1 is a local hardened v1 security core, not a production-ready enterprise product.

Current baseline: runtime PASS, parser bypass corpus PASS, policy regression 31/31 PASS, approval regression PASS, audit-chain restart continuity PASS, corrupt audit startup fail-fast PASS, chaos/failure-mode suite 10/10 PASS.

Known gaps remain: auth/TLS/RBAC, metrics/observability, log rotation, full historical audit-chain verification, distributed audit-chain support, long-running soak testing, and production deployment hardening.
```
