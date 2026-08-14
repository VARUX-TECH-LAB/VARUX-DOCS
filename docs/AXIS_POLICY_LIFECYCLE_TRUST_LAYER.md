# AXIS Policy Lifecycle & Trust Layer

AXIS v0.5 adds controlled policy lifecycle management around the existing deterministic policy engine. The write-path enforcement model is unchanged: production SQL still flows through the classifier, policy evaluator, approval store, executor, and audit evidence path already used by AXIS.

The lifecycle layer exists to make policy change a security-sensitive operation instead of a direct edit to production behavior.

## What v0.5 Adds

- Candidate policy validation before activation.
- Active-versus-candidate policy diffing.
- Dry-run decision previews for current or candidate policy.
- Immutable policy version files with a manifest.
- Safe candidate activation with expected-hash checks.
- Rollback to a previous valid stored policy.
- Control-plane views for validation, diff, dry-run, versions, activation, and rollback.
- Optional operator token protection for mutating lifecycle endpoints.

## Why Policy Lifecycle Matters

A policy file controls the production write path. A bad policy can accidentally allow broad writes, remove approval gates, or weaken DDL restrictions. AXIS v0.5 makes each policy change answer these questions:

- Is the policy structurally valid?
- Does it contain risky semantic patterns?
- What changed from the active policy?
- What would representative SQL do under the candidate?
- Is the exact stored candidate hash being activated?
- Can the operator roll back to a known-good version?

## Validation Model

`POST /policy/validate` validates a supplied JSON policy without mutating runtime state.

Structural checks include:

- Policy is a JSON object.
- `policy_version` or `version` exists.
- `write_rules` exists and is an array.
- Rule IDs are present and unique.
- Action values are valid: `ALLOW`, `BLOCK`, `REQUIRE_APPROVAL`, or `APPROVAL_REQUIRED`.
- Required typed fields deserialize through the existing policy model.

Semantic checks currently warn on:

- Dangerous production wildcard `ALLOW` rules.
- Default write or DDL behavior that is too permissive.
- Unknown SQL shape fallback behavior that is not a hard block.
- Missing DELETE-without-WHERE, batch UPDATE, batch DELETE, or DDL guards.
- Duplicate match conditions.
- Shadowed rules where an earlier rule can match before a later one.
- Broad migration-oriented production allow rules.
- Unknown environment labels.

Validation is read-only. It does not reload policy, execute SQL, create approvals, or write production audit evidence.

## Diff Model

`POST /policy/diff` compares the active policy with a candidate policy.

The diff reports:

- Added rules.
- Removed rules.
- Modified rules.
- Action changes.
- Match scope changes.
- Approval and reason-code changes.
- Risk level for each change and for the whole diff.

Rules are matched by rule ID. If IDs are absent, AXIS falls back to a normalized JSON fingerprint. Stable rule IDs are recommended for operator-readable diffs.

Risk is conservative. Removing production protections or changing `BLOCK` / `REQUIRE_APPROVAL` to `ALLOW` is treated as higher risk.

## Dry-Run Model

`POST /policy/dry-run` previews a decision under either the current active policy or a supplied candidate policy.

Dry-run reuses the existing SQL classifier and policy evaluator. It returns:

- Decision.
- Reason.
- SQL fingerprint.
- Classification details.
- Matched rule trace.
- `would_execute: false`.
- `audit_written: false`.

Dry-run never executes SQL, never writes audit evidence, and never creates approval requests. Unsupported or unparsable SQL fails safe with a blocking preview.

## Version Store Model

The default local store is:

```text
data/policies/
  active.json
  manifest.json
  versions/
    policy-vYYYYMMDD-HHMMSS-xxxxxxxx.json
```

Each immutable version document contains:

- Version record metadata.
- The policy JSON.
- The validation result at creation time.

The manifest tracks:

- Version ID.
- Policy version string.
- Hash.
- Created and activated timestamps.
- Status: `active`, `candidate`, `archived`, or `rejected`.
- Rule count.
- Validation status.
- Source.
- Optional operator note.

Version files are created with create-new semantics and are not rewritten. The manifest and active pointer are updated through temp-file writes and replacement with backup recovery where the platform allows it.

## Activation Safety Model

`POST /policy/activate` activates a stored candidate only when:

- The candidate exists.
- Its status is `candidate`.
- `expected_hash` matches the stored hash.
- The stored policy file hash still matches the manifest hash.
- The policy validates again immediately before activation.

Activation writes `active.json`, updates the manifest status, and swaps the in-memory active policy handle. If validation or hash verification fails, AXIS keeps the previous active policy.

AXIS does not delete old versions during activation.

## Rollback Model

`POST /policy/rollback` activates a previous valid stored version by ID or policy version string.

Rollback requires:

- Target version exists.
- Target is not rejected.
- `expected_hash` matches.
- Target policy validates before activation.

The current active version becomes archived and the target becomes active.

## Operator Token Behavior

AXIS v0.5 supports a minimal optional operator token:

```text
AXIS_OPERATOR_TOKEN=...
```

When configured, mutating lifecycle endpoints require:

```text
X-AXIS-Operator-Token: ...
```

Protected endpoints:

- `POST /policy/candidates`
- `POST /policy/activate`
- `POST /policy/rollback`

Read-only lifecycle endpoints remain open for local visibility. If no token is configured, local development remains unblocked and `/policy/status` reports `operator_auth_enabled: false`.

## Intentionally Not Included

AXIS v0.5 does not include:

- Full RBAC.
- SaaS control plane.
- Remote policy distribution.
- Multi-instance consensus.
- Automatic policy optimization.
- AI-generated policy activation.

Those are future trust and deployment concerns. v0.5 focuses on safe local lifecycle control for one AXIS runtime.
