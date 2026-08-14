# Reviewer Attack Checklist

Selected reviewers are invited to challenge AXIS deliberately. The goal of this package is technical damage discovery before broader executive or CISO outreach.

## Currently Covered By The Pilot

| Attack idea | Current evidence | Current status |
|---|---|---|
| Dangerous DELETE without WHERE | `demo/evidence/pilot-v1/responses/blocked-operation-response.json` | Covered for the pilot endpoint and policy rule. |
| ORM-generated risky update | `demo/evidence/pilot-v1/responses/approval-required-response.json` | Covered for role update requiring approval. |
| Approval replay attempt | `demo/evidence/pilot-v1/responses/approval-rejected-response.json` and approval retry flow | Partly covered through rejected approval retry behavior; broader replay cases need more tests. |
| Retry after rejected approval | `demo/evidence/pilot-v1/responses/approval-rejected-response.json` | Covered for the pilot flow. |
| Transaction rollback verification | `demo/evidence/pilot-v1/responses/transaction-approval-rollback-response.json`, `transaction-blocked-rollback-response.json` | Covered for two pilot workflows. |
| Audit evidence inspection | `demo/evidence/pilot-v1/audit/audit-sample-events.json` | Covered for captured protected operations. |

## Partially Covered

| Attack idea | What to inspect | Current status |
|---|---|---|
| Multi-statement SQL bypass attempt | Policy classifier and parser paths under `src/` plus pilot policy | Some parser hardening exists, but this package does not claim all multi-statement bypass classes are closed. |
| Dangerous UPDATE without WHERE | Policy rules and parser cases | Policy concept is present; the reviewer should test variants against current rules. |
| DDL attempt | Policy rules and destructive-operation handling | DDL blocking is in policy scope, but the pilot evidence focuses on selected business endpoints. |
| Malformed request handling | `src/gate/`, `src/errors.rs`, sample backend response mapping | Structured outcomes are shown; malformed-input coverage should be expanded by reviewers. |
| Policy metadata inspection | `demo/evidence/pilot-v1/audit/audit-sample-events.json` | Audit events include metadata; completeness should be reviewed. |

## Known Limitation

| Attack idea | Limitation |
|---|---|
| Direct DB bypass discussion | The pilot uses application-layer SQLAlchemy routing. Direct write bypass prevention requires DB role separation, credential control, and network policy. |
| Backend direct DB credential misuse discussion | The app still has direct PostgreSQL connectivity for safe reads. Preventing misuse is a deployment-hardening problem not fully proven here. |
| Missing/corrupt audit evidence scenario | The package shows audit generation and endpoint verification, but does not include independent offline recomputation for every event. |
| Policy mismatch scenario | The pilot does not prove every production policy drift or mismatch case is detected. |

## Future Work

- Native PostgreSQL wire/proxy enforcement.
- Stronger direct-write credential isolation.
- Expanded parser and multi-statement adversarial corpus.
- Broader ORM/framework coverage.
- Independent offline audit hash verification.
- Controlled benchmark work separate from this reviewer package.

Do not treat this checklist as a claim that all listed attack paths are solved.
