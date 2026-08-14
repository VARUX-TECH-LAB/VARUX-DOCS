# AXIS Failure Mode Matrix

AXIS v1 is not a production-ready enterprise product. It is a local hardened v1 security core. This matrix documents the tested local failure-mode baseline and the expected safety property for each case.

## Matrix

| Failure mode | Test source | Expected behavior | Verified result | Remaining risk |
| --- | --- | --- | --- | --- |
| PostgreSQL down | `axis_chaos_test.py::postgres_down_fail_safe` | Risky writes must not receive unsafe `200 ALLOW`; service recovers after PostgreSQL restart. | PASS | Availability is degraded while DB is down; no external alerting. |
| Audit path unwritable | `axis_chaos_test.py::audit_unwritable_no_write_execution` | Writes must not execute when audit evidence cannot be written. | PASS | No automated remediation or disk-space monitoring. |
| Invalid policy JSON | `axis_chaos_test.py::invalid_policy_startup_fail_fast` | AXIS must fail startup or become unhealthy instead of running with invalid policy. | PASS | No signed policy or centralized policy distribution. |
| Corrupt approval store | `axis_chaos_test.py::approval_store_corrupt_fail_safe` | AXIS must fail safely; corrupt approval state must not allow writes. | PASS | Recovery is manual; no approval log compaction or repair tool. |
| Malformed JSON request | `axis_chaos_test.py::malformed_json_controlled_rejection` | Bad request payloads must receive controlled client errors. | PASS | No WAF/rate-limit layer. |
| Huge payload | `axis_chaos_test.py::huge_payload_survives` | Oversized or pathological payloads must not produce unsafe allow and must not permanently break service. | PASS | Long-running resource exhaustion testing is not complete. |
| Concurrent approval race | `axis_chaos_test.py::concurrent_approval_resolve_race` | A single approval must not execute more than once; only one final decision should win. | PASS | Single-process local store only; no distributed lock model. |
| Restart during traffic | `axis_chaos_test.py::restart_during_traffic` | Risky SQL must not receive unsafe `200 ALLOW` during restart pressure. | PASS | No rolling-deploy or multi-instance behavior validated. |
| DB pool pressure | `axis_chaos_test.py::db_pool_pressure` | Pool pressure must not produce unsafe allow for blocked or approval-required requests. | PASS | No production SLO or saturation metrics. |
| Corrupt final audit entry at startup | `axis_chaos_test.py::corrupt_audit_startup_fail_fast` | AXIS must fail startup when the final audit entry cannot be parsed for chain recovery. | PASS | Full historical chain verification is not implemented. |
| Audit-chain restart continuity | `axis_audit_restart_test.py` | First post-restart event `previous_hash` must equal pre-restart last `event_hash`. | PASS | Verifies continuity at restart boundary only, not whole history. |
| Parser bypass corpus | `axis_regression.py` with `tests/parser_bypass_cases.json` | Risky parser bypass cases must not result in unsafe `200 ALLOW`. | PASS | Corpus is finite and should keep growing. |
| Policy regression | `axis_regression.py` with `tests/policy_cases.json` | Read/write/block/approval decisions must match policy expectations. | PASS | Policy coverage is for current `prod_main` baseline only. |
| Approval regression | `axis_regression.py` with `tests/approval_cases.json` | Approval-required rules must create approval flow outcomes. | PASS | No authenticated approver identity in v1. |
| Stress baseline | `axis_gate_stress.py` | 1000 requests at 50 concurrency must finish with unexpected result count 0. | PASS | Not a long-running soak test. |

## Failure Policy

The v1 safety policy is fail-safe for write paths:

- If SQL cannot be parsed or classified safely, reject it.
- If policy cannot be loaded, do not start normally.
- If audit evidence cannot be written, do not execute writes.
- If approval state is corrupt, do not approve or execute from that state.
- If PostgreSQL is unavailable, do not convert risky writes into `ALLOW`.

## Not Yet Covered

- Authenticated approver identity failure modes.
- RBAC denial paths.
- Log rotation and retention failures.
- Full audit-chain replay verification failures.
- Multi-instance split-brain or distributed audit-chain conflicts.
- Long-duration soak, disk pressure, memory pressure, and process supervisor behavior.
