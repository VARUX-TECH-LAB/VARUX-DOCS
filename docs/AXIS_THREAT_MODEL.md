# AXIS Threat Model

## Assets Protected

- Production database write path.
- Policy integrity.
- Audit evidence integrity.
- Approval workflow integrity.
- Operator control plane.
- Policy lifecycle store.

## Threat Actors

- Compromised application credential.
- Buggy internal tool.
- Malicious insider.
- Careless operator.
- Compromised migration script.
- Attacker with partial network access.
- Attacker trying to tamper with logs.
- Attacker trying to weaken policy.

## Threat Scenarios

| Scenario | Risk | Current AXIS response |
| --- | --- | --- |
| A. Destructive `DELETE` reaches production | Data loss | SQL classification detects `DELETE`, risk signals include `delete_without_where` when applicable, policy can `BLOCK` or require approval |
| B. `UPDATE` without `WHERE` | Broad unintended mutation | Scope estimation marks batch writes; default policy requires approval for batch updates |
| C. DDL in production | Schema damage or outage | DDL classification and default DDL policy require approval |
| D. Hidden mutation through unsupported SQL shape | Parser bypass | Unsupported or write-like SQL shapes fail safe instead of being treated as safe reads |
| E. Approval bypass attempt | Risky write executes before review | `REQUIRE_APPROVAL` creates a pending approval and does not execute until resolution |
| F. Audit log tampering | Evidence cannot be trusted | Hash-chain verification reports malformed records, missing hashes, and broken links |
| G. Policy downgrade | Broad writes become allowed | Validation and diff highlight risky changes before activation |
| H. Candidate policy activated without validation | Invalid policy controls write path | Candidate creation and activation validate policy; invalid candidates are rejected |
| I. Rollback to malicious policy | Old bad policy becomes active | Rollback validates target policy and checks expected hash |
| J. Operator endpoint misuse | Unauthorized lifecycle mutation | Optional `AXIS_OPERATOR_TOKEN` protects candidate creation, activation, and rollback |
| K. Frontend/backend mismatch | Operator sees stale or wrong state | Control plane reads live endpoints in real mode and surfaces backend errors |
| L. Local policy store corruption | Runtime loads inconsistent policy | Store reads validate records and policy activation keeps previous active policy on failure paths |

## Mitigations Currently Implemented

- Policy enforcement for `ALLOW`, `BLOCK`, and `REQUIRE_APPROVAL`.
- Unsupported SQL fail-safe behavior through parser/classifier rejection.
- Approval workflow that records pending requests and prevents execution until approval.
- Audit hash chain with `previous_hash` and `event_hash`.
- Restart continuity from the last recorded audit event hash.
- Evidence verification through `GET /evidence/verify`.
- Policy validation for structural and semantic policy issues.
- Policy diff to highlight added, removed, modified, and risky rules.
- Policy dry-run that previews decisions without execution or mutation.
- Expected-hash activation for stored candidates.
- Rollback validation and expected-hash checks.
- Optional operator token for mutating lifecycle endpoints.

## Known Gaps

- No full RBAC.
- No SSO.
- No distributed consensus.
- No external KMS.
- No tamper-proof external ledger.
- No formal compliance mapping.
- Local JSON/file-backed policy store.
- No dedicated operator audit stream yet.
- Request identity fields are not cryptographically verified by AXIS itself.
- No multi-instance policy synchronization.

## Pilot Risk Posture

AXIS v0.6 is suitable for:

- Local evaluation.
- Controlled demo.
- Technical review.
- Non-production pilot planning.
- Staging or lab validation against representative SQL flows.

AXIS v0.6 is not yet suitable for:

- Unsupervised production deployment.
- Regulated deployment without additional controls.
- Multi-region enterprise rollout.
- Deployment where AXIS is exposed without network, identity, and transport protections.

## Review Focus

Reviewers should pay particular attention to:

- SQL parser and classifier coverage for their query patterns.
- Whether policy defaults are conservative enough for the target environment.
- Whether approval records and audit evidence meet local evidence requirements.
- Whether local file-backed state is acceptable for a pilot.
- Which identity, TLS, KMS, retention, and operator audit controls must exist before production use.
