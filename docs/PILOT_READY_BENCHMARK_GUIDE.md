# AXIS Pilot Benchmark Guide

## 1. Performance Philosophy: Measurement over Assumption

AXIS does not compete with raw database latency.
AXIS competes with uncontrolled production execution risk.

The pilot benchmark measures latency honestly by separating semantically different paths. It does not collapse safe reads/writes, approval-required work, and blocked dangerous SQL into one average latency number. AXIS may add measurable overhead. That overhead is evaluated against deterministic production write-path control, approval safety, auditability, and signed evidence generation.

## 2. Data Paths

Direct Path:

Application / Agent -> PostgreSQL

AXIS HTTP Path:

Application / Agent -> AXIS Execution Gateway (/query API) -> Policy Engine -> PostgreSQL

AXIS Approved Path:

Application / Agent (+approval_id) -> AXIS Execution Gateway -> CAS State Verification -> PostgreSQL

Native Wire Path:

Application / Agent -> AXIS Native Proxy -> PostgreSQL

The Control Plane is a management path, not the data path. Benchmark reports must not describe query execution as `Application -> AXIS Control Plane -> PostgreSQL`.

## 3. Benchmark Corpus Split

The default pilot benchmark uses a 70/20/10 corpus split:
70% clean allow-path queries, 20% approval-required controlled-risk queries, and 10% blocked dangerous or ambiguous queries.

The split is configurable with:

```powershell
python scripts/baseline_benchmark.py ^
  --queries 10000 ^
  --allow-ratio 70 ^
  --approval-ratio 20 ^
  --block-ratio 10 ^
  --output-format markdown,json
```

The benchmark validates that ratios sum to 100 and that `--queries` is greater than zero. A fixed `--seed` makes corpus selection deterministic.

## 4. Why Direct PostgreSQL Comparison Uses Only ALLOW Corpus

Direct PostgreSQL comparison is performed only against the allow-path corpus. Approval-required and blocked queries are measured as AXIS enforcement paths, because their purpose is to prevent unsafe execution rather than compete with raw database execution latency.

Dangerous SQL is never sent directly to PostgreSQL by the benchmark. DROP, TRUNCATE, blind DELETE, blind UPDATE, multi-statement write chains, unsupported SQL, malformed SQL, and unresolved prepared statements are sent only through AXIS, where they must fail closed before database execution.

## 5. Metrics Collected

Section 1 compares only Direct PostgreSQL ALLOW Path and AXIS HTTP `/query` ALLOW Path:

- p50, p95, and p99 latency
- min, max, and average latency
- throughput QPS
- error rate
- total, successful, and failed queries

Section 2 measures AXIS-only enforcement behavior for `REQUIRE_APPROVAL` and `BLOCK`:

- p50, p95, and p99 decision latency
- average decision latency
- approval record creation success count
- blocked count
- structured error count
- error rate

Section 3 summarizes security outcomes:

- total corpus size
- ALLOW, REQUIRE_APPROVAL, and BLOCK counts
- queries executed against DB
- queries not executed
- approval records created
- blocked before DB
- failed-closed count
- policy_id, policy_version, and policy_sha256 when available

## 6. Human Approval Time Exclusion

The approval-required measurement stops when AXIS returns `REQUIRE_APPROVAL`, creates the approval record, and reports `execution_status=NOT_EXECUTED`. Human wait time is not included.

Approved retry execution, when tested, is a separate path:

Application / Agent (+approval_id) -> AXIS Execution Gateway -> CAS State Verification -> PostgreSQL

That retry path must not be mixed into the ALLOW baseline comparison.

## 7. Native Wire Path Handling

Native Wire Path is optional. The benchmark checks it only when `--native-wire-check` is passed.

If a real native proxy is not enabled and reachable, the report marks:

```text
Native Wire Path: SKIP
Reason: native wire proxy is not enabled in this build.
```

The benchmark must not fake native wire behavior, SQLSTATE values, or support claims.

## 8. Report Interpretation

The Markdown report is written to:

```text
demo/benchmark-results/axis-benchmark-report.md
```

The JSON report is written to:

```text
demo/benchmark-results/axis-benchmark-report.json
```

Read the report in three sections. Section 1 answers the fair latency question for safe operations. Section 2 answers how quickly AXIS makes enforcement decisions. Section 3 answers what AXIS prevented, suspended, or allowed.

Do not average Section 1 and Section 2 together. A blocked DROP TABLE and a direct `SELECT 1` are not competing operations.

## 9. Objection Handling

Objection: "Why not compare blocked SQL to PostgreSQL?"

Answer: because direct execution of blocked SQL would damage or mutate the database. Timing a direct DROP, TRUNCATE, blind DELETE, or ambiguous multi-statement write is not benchmarking. It is self-sabotage with timing numbers.

Objection: "AXIS adds latency."

Answer: latency is measured directly in Section 1. The pilot question is whether that measured overhead is acceptable for deterministic production write-path control, approval safety, auditability, and verifiable evidence.

Objection: "Approval-required operations are slower than direct PostgreSQL."

Answer: approval-required operations are intentionally not direct database operations. Their purpose is to suspend controlled-risk work, create durable approval evidence, and prevent execution until an authorized retry supplies an approval id.

Objection: "Blocked operations have no PostgreSQL baseline."

Answer: correct. Their success condition is non-execution before the database, with structured evidence. Raw database latency is not the relevant comparison for a prevented unsafe operation.

## 10. Running the Benchmark

Default pilot run:

```powershell
python scripts/baseline_benchmark.py ^
  --queries 10000 ^
  --allow-ratio 70 ^
  --approval-ratio 20 ^
  --block-ratio 10 ^
  --output-format markdown,json
```

Common local run:

```powershell
python scripts/baseline_benchmark.py --queries 100 --output-format markdown,json
```

Useful options:

- `--axis-url http://localhost:6543`
- `--pg-host localhost`
- `--pg-port 5432`
- `--pg-user varux`
- `--pg-password varux`
- `--pg-database prod_main`
- `--tenant acme_corp`
- `--env production`
- `--actor benchmark_runner`
- `--actor-type service`
- `--output-dir ./demo/benchmark-results`
- `--concurrency 1`
- `--warmup 100`
- `--seed 42`
- `--native-wire-check`

If AXIS is not reachable, the benchmark reports an environment error and exits without treating that as benchmark failure:

```text
[ENVIRONMENT ERROR] AXIS is not reachable at <axis-url>.
This is not a benchmark failure. Start AXIS and rerun.
```

If PostgreSQL is not reachable, the benchmark reports:

```text
[ENVIRONMENT ERROR] PostgreSQL is not reachable at <host:port>.
This is not an AXIS failure. Check database service and credentials.
```

Final proof sentence:

Latency is measured.
Risk reduction is demonstrated.
Evidence is verifiable.
