# AXIS Security Classification Contract (SCC) v1.0

**Document Type:** Engineering Binding Contract  
**Scope:** Classifier & Policy Evaluator  
**Status:** ACTIVE – Locked  
**Violation Severity:** Critical (Authorization Bypass)  
**Effective Date:** 2026-08-07  
**Supersedes:** All prior implicit classification heuristics  
**Amendment Authority:** Security Architecture Team only

---

## 1. Purpose

This contract defines the **authoritative classification rules** for every SQL statement processed by AXIS. The Classifier must produce `QueryType` and `Operation` based on **real PostgreSQL semantic effect**, not on outer syntax. The Policy Evaluator treats this classification as the **single source of truth** for authorization decisions.

Any deviation from this contract is a **Critical Security Defect** and must be treated as an active Authorization Bypass.

---

## 2. Core Principles

These principles are non-negotiable and derived from AXIS's existing security model:

| Principle | Definition |
|-----------|------------|
| **Semantic > Syntax** | Classification must reflect the actual database state change, not the first keyword. |
| **State Mutation = Write** | Any statement that creates, alters, deletes, or moves persistent data or metadata is `Write`. |
| **Fail-Closed on Ambiguity** | If semantic effect cannot be determined from AST, default to `Write` or `Block`. Never default to `Read` unless proven read-only. |
| **Defense in Depth** | Risk signals are decision inputs, not decorative metadata. They can override a permissive read default **only as a fallback** when the Classifier is incorrect. |
| **Deterministic** | Same SQL under same context must always produce same classification. |
| **CI-Enforced** | All rules in this contract must be enforced by automated tests in CI. Human review alone is insufficient. |

---

## 3. Mandatory Classification Rules (Primary)

### 3.1. Read (QueryType::Read)

**Condition:** Statement does **not** modify any persistent or session state. It only returns data.

**Examples (non-exhaustive):**
- `SELECT ...` (without INTO, without volatile functions, without data-modifying CTEs)
- `SHOW`, `EXPLAIN`, `DESCRIBE`
- `COPY TO STDOUT`
- `SELECT` with `IMMUTABLE` or `STABLE` functions only
- `LISTEN` / `NOTIFY` (state change is minimal; treat as Read for classification, but audit separately)

---

### 3.2. Write (QueryType::Write) – PRIMARY CLASSIFICATION

**Condition:** Statement creates, alters, deletes, or moves persistent data, schema, sequences, or large objects. This includes any statement that would change the result of a subsequent `SELECT` on the same table.

**Mandatory Write Patterns (Minimum list – must be classified as Write):**

| SQL Pattern | Example | Operation |
|-------------|---------|-----------|
| `INSERT` | `INSERT INTO users ...` | DML |
| `UPDATE` | `UPDATE users SET ...` | DML |
| `DELETE` | `DELETE FROM users ...` | DML |
| `SELECT INTO` | `SELECT * INTO archive FROM users` | DDL + DML |
| `CREATE TABLE AS` | `CREATE TABLE archive AS SELECT * FROM users` | DDL + DML |
| `MERGE` | `MERGE INTO users USING ...` | DML |
| `TRUNCATE` | `TRUNCATE users` | DDL |
| `DROP`, `ALTER` | `DROP TABLE users`, `ALTER TABLE ...` | DDL |
| `REFRESH MATERIALIZED VIEW` | `REFRESH MATERIALIZED VIEW mv` | DML |
| `COPY FROM` | `COPY users FROM STDIN` | DML |
| `COPY TO PROGRAM` | `COPY users TO PROGRAM 'cmd'` | Critical – always BLOCK |
| `CALL` (procedure) | `CALL proc()` | DML (unless proved read-only) |
| `DO` (anonymous block) | `DO $$ ... $$` | DML (unless proved read-only) |
| `nextval()`, `setval()` | `SELECT nextval('seq')` | Sequence mutation |
| `lo_import`, `lo_export` | `SELECT lo_import('/path')` | Large object mutation |
| `ALTER SYSTEM` | `ALTER SYSTEM SET ...` | Critical – always BLOCK |
| Data-modifying CTE | `WITH d AS (DELETE ...) SELECT * FROM d` | DML (write happens before outer read) |
| Nested/Chained CTE writes | `WITH d1 AS (UPDATE ...), d2 AS (DELETE ...) SELECT *` | DML |

---

### 3.3. Unknown / Ambiguous (QueryType::Unknown)

**Condition:** AST does not provide enough information to determine state mutation (e.g., function call without volatility metadata, external procedure).

**Mandatory Behavior:**
- If `VOLATILE` or `SECURITY DEFINER` → treat as **Write** or **Block** unless explicitly allowlisted.
- If function volatility is unknown → treat as **Write** (fail-closed).
- If procedure (`CALL`) or anonymous block (`DO`) → treat as **Write**.
- If `COPY PROGRAM` → treat as **Block** (always).

---

## 4. Risk Signals – Defense in Depth (Secondary, Not Primary)

**IMPORTANT:** This section is a **fallback mechanism**, not the primary security control. The Classifier must first correctly classify according to Section 3. The risk signal override exists **only** to prevent a bypass if the Classifier fails.

### 4.1. Mandatory Risk Signals (for Write patterns)

| Risk Signal | Trigger Condition | Override Behavior |
|-------------|-------------------|-------------------|
| `select_into` | `SELECT INTO` detected | If `QueryType == Read` (Classifier error), override to `Block` or `RequireApproval`. |
| `cte_write` | Data-modifying CTE detected | If `QueryType == Read` (Classifier error), override to `Block` or `RequireApproval`. |
| `sequence_write` | `nextval()` / `setval()` detected | If `QueryType == Read` (Classifier error), override to `Block` or `RequireApproval`. |
| `volatile_function` | `VOLATILE` function call detected | If `QueryType == Read` (Classifier error), override to `Block` or `RequireApproval`. |
| `copy_program` | `COPY TO PROGRAM` detected | Always `Block`. |

### 4.2. Policy Evaluator Behavior

The Evaluator **must**:
1. Read `QueryType` from Classifier.
2. Read `RiskSignals` from Classifier.
3. **Primary path:** If `QueryType == Write`, apply `defaults.write` rules directly.
4. **Fallback path:** If `QueryType == Read` **and** a critical risk signal exists, log a warning (`CLASSIFIER_OVERRIDE_TRIGGERED`) and apply the override rule.
5. **Never** rely on the override as the primary protection. The primary protection is correct `QueryType` from the Classifier.

---

## 5. Testing Requirements (CI-Enforced)

This contract is **not valid** until proven by automated tests in CI. Human review alone is insufficient.

### 5.1. Positive Tests (Write classification)

For every mandatory Write pattern listed in Section 3.2, at least one test must assert:
- `QueryType == Write`
- `RiskSignals` contain the expected signal (if applicable)

**CI Rule:** If a new Write pattern is added to the parser but no corresponding test exists, the PR must fail.

### 5.2. Negative Tests (Read classification)

For every pure read pattern listed in Section 3.1, at least one test must assert:
- `QueryType == Read`
- No critical risk signal is present

### 5.3. Override Tests (Defense in Depth)

For at least `SELECT INTO` and CTE write:
- Set `defaults.read = ALLOW`
- Send the query
- Assert decision is **BLOCK** or **REQUIRE_APPROVAL** due to risk signal override

### 5.4. Mutation Tests (Survival Proof)

For at least one critical detection (e.g., `IntoClause` detection):
- Disable the detection logic in code
- Run the corresponding test
- Assert test **FAILS** (red)
- Restore logic → test **PASSES** (green)

**CI Rule:** This must run in CI. If it ever fails, the build breaks.

---

## 6. Governance

- **Amendments:** Any change to this contract requires Security Architecture Team approval **and** a corresponding update to the CI test suite.
- **Violations:** Any code merged that violates this contract is a **Critical Security Defect** and must be reverted immediately.
- **New PostgreSQL Features:** Before supporting a new PostgreSQL syntax, the feature must be evaluated against this contract and tests added. The PR must include the new tests.
- **Audit Trail:** Every change to classification logic must reference the contract rule it implements.
- **CI Enforcement:** The CI pipeline must:
  - Run all tests from Section 5.
  - Fail if any test is missing for a newly added SQL construct.
  - Fail if mutation tests (Section 5.4) pass when detection logic is disabled.
  - Block PR merge until all tests pass.

---

## 7. Sign-off

| Role | Name | Date | Signature |
|------|------|------|-----------|
| Security Architect | | 2026-08-07 | ✅ |
| Engineering Lead | | 2026-08-07 | ✅ |
| Product Architect | | 2026-08-07 | ✅ |

---

## 8. Final Declaration

**This contract is the authoritative source for classification behavior. Code that violates it is not AXIS-compliant.**

**The primary security control is correct QueryType from the Classifier (Section 3).**

**Risk signals (Section 4) are a fallback defense-in-depth mechanism, not a substitute for correct classification.**

**CI enforces all rules automatically. Human review is not sufficient.**

---

**Status:** ✅ LOCKED – Active and Enforceable  
**Next Action:** Implement in code. Write tests. Run CI. Merge.

---