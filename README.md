# VARUX AXIS — Documentation Site

This repository hosts the **VARUX AXIS** technical documentation as a static HTML site (no build step, no dependencies — open any `.html` file in a browser).

**AXIS** is a deterministic PostgreSQL write-path control layer: it classifies SQL, evaluates versioned policy (ALLOW / BLOCK / REQUIRE_APPROVAL), and records durable, hash-linked evidence — no protected production write without a paper trail.

## Contents

| File | Page |
| --- | --- |
| `index.html` | Overview (start here) |
| `getting-started.html` | Getting Started |
| `installation.html` | Installation |
| `configuration.html` | Configuration |
| `architecture.html` | Architecture |
| `request-lifecycle.html` | Request Lifecycle |
| `sql-classification.html` | SQL Classification |
| `policy-engine.html` | Policy Engine |
| `approval-workflow.html` | Approval Workflow |
| `audit-evidence.html` | Audit & Evidence |
| `operating-modes.html` | Operating Modes |
| `security-model.html` | Security Model |
| `mtls-network.html` | mTLS & Network |
| `transactions.html` | Transactions |
| `orm-extended-query.html` | ORM & Extended Query |
| `native-pg-protocol.html` | Native PG Wire Protocol |
| `api-reference.html` | API Reference |
| `error-codes.html` | Error Codes |
| `deployment.html` | Deployment |
| `operations.html` | Operations |
| `failure-recovery.html` | Failure & Recovery |
| `troubleshooting.html` | Troubleshooting |
| `testing-verification.html` | Testing & Verification |
| `security-review.html` | Security Review |
| `limitations.html` | Limitations |
| `pilot.html` | Pilot |
| `faq.html` | FAQ |
| `glossary.html` | Glossary |
| `changelog.html` | Changelog |
| `docs.html` | Combined source document |
| `axis-map.html` | AXIS explainer visual map |

## Serve

GitHub Pages: **Settings → Pages → Source: main / root** → <https://varux-tech-lab.github.io/AXIS-DOCS/>

Or serve locally:

```
python -m http.server 8000
# http://localhost:8000
```

## Status

- **v0.6 — Pilot Readiness:** controlled demos, local review, non-production pilot.
- This documentation is engineering material, **not** a compliance certification.

## Links

- Main site: <https://varuxcyber.com>
- AXIS product page: <https://varuxcyber.com/axis>
