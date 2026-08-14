# AXIS Reviewer Quickstart

## Purpose

The AXIS reviewer demo verifies core security behavior on a clean machine with Docker:

- policy enforcement
- dangerous operation blocking
- approval routing
- audit evidence generation
- database state preservation after blocked operations

## Command

```sh
docker compose -f demo/docker-compose.reviewer.yml up --build
```

The reviewer client runs inside Docker. Do not install Python packages, npm packages, Rust tools, psql, or other local dependencies for this demo.

## Reviewer URLs

AXIS:

```text
http://localhost:65430
```

PostgreSQL:

```text
localhost:54320
```

Control Plane:

```text
Not included in demo/docker-compose.reviewer.yml. If a reviewer UI service is added later, it must use http://localhost:30000.
```

The reviewer compose file does not use host ports `5432`, `3000`, or `6543`.

## Evidence Output

Evidence files are written to:

```text
./demo-evidence-output
```

Expected files:

- `audit.log`
- `audit.wal`
- `approvals.sqlite`
- `reviewer-demo-report.json`
- `reviewer-demo-report.md`
- `evidence-bundle/audit-export.json`

## Success Condition

The demo is successful only if the `reviewer-client` container exits with code `0` and prints:

```text
AXIS REVIEWER DEMO: PASS
```

## Failure Condition

If any check fails, `reviewer-client` exits with code `1` and prints:

- failed check name
- expected result
- actual result
- short explanation

The JSON and Markdown reports in `./demo-evidence-output` contain the same failure details.
