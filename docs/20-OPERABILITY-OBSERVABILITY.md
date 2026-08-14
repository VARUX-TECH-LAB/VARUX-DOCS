---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# Operability and Observability

## Purpose

This document defines metrics, logs, traces, dashboards, health checks, diagnostics, and on-call workflows required for AXIS Native PG mode.

AXIS is not operable if an on-call engineer cannot answer within minutes:

- why was this query blocked?
- did it reach backend?
- which policy version decided?
- is audit healthy?
- is the connection state safe?
- are we in enforcement, audit-only, bypass, or degraded mode?

## Required Metrics

### Connection Metrics

- `axis_active_client_connections`
- `axis_active_backend_connections`
- `axis_connection_accept_rate`
- `axis_connection_reject_rate`
- `axis_connection_close_reason_total`
- `axis_backend_connect_errors_total`
- `axis_connection_poisoned_total`

### Decision Metrics

- `axis_policy_allow_total`
- `axis_policy_block_total`
- `axis_policy_approval_required_total`
- `axis_policy_eval_latency_seconds_bucket`
- `axis_policy_eval_queue_depth`
- `axis_policy_timeout_total`

### Audit Metrics

- `axis_audit_append_latency_seconds_bucket`
- `axis_audit_fsync_latency_seconds_bucket`
- `axis_audit_queue_depth`
- `axis_audit_circuit_breaker_state`
- `axis_audit_wal_bytes_total`
- `axis_audit_wal_disk_free_bytes`
- `axis_audit_export_total`
- `axis_audit_verify_failure_total`

### PostgreSQL Protocol Metrics

- `axis_simple_query_total`
- `axis_extended_query_rejected_total`
- `axis_copy_blocked_total`
- `axis_cancel_request_total`
- `axis_unsupported_protocol_total`
- `axis_ready_for_query_state_total{state="I|T|E"}`
- `axis_command_complete_total`

### Failure Metrics

- `axis_execution_unknown_total`
- `axis_backend_confirmed_client_delivery_unknown_total`
- `axis_safety_rollback_attempt_total`
- `axis_safety_rollback_failed_total`
- `axis_restart_recovery_incomplete_total`
- `axis_bypass_declared_total`

## High-Cardinality Guardrails

Do not use the following as Prometheus labels:

- `axis_request_id`
- raw SQL
- SQL fingerprint if high-volume/unbounded
- raw actor ID
- ticket ID
- client IP if high-cardinality in multi-tenant deployments
- table names unless bounded and explicitly accepted

Use logs/traces for request-scoped values. Metrics must survive production cardinality; otherwise Prometheus becomes a bonfire with an HTTP port.

## Required Structured Logs

Every decision log should include:

- timestamp;
- level;
- event_type;
- axis_request_id;
- session_id;
- policy_id;
- policy_version;
- decision;
- risk;
- backend_forwarded;
- tcp_bytes_forwarded;
- transaction_state;
- actor context where safe;
- error class;
- evidence hash pointer.

Raw bind values and raw sensitive SQL must not be logged by default.

## Required Traces

OpenTelemetry spans:

```text
axis.session
  axis.startup
  axis.query.receive
  axis.classify
  axis.policy.evaluate
  axis.audit.intent_append
  axis.backend.forward
  axis.backend.response
  axis.audit.completion_append
  axis.client.respond
```

For rollback:

```text
axis.safety_rollback.issue
axis.safety_rollback.result
```

For approval:

```text
axis.approval.lookup_or_create
axis.approval.response
```

Span links should connect approval ticket creation to retry execution.

## Required Dashboards

### Operator Overview

- enforcement mode;
- audit health;
- backend health;
- policy version;
- active connections;
- decision rates;
- p95/p99 policy and audit latency;
- current failure count.

### Query Forensics

Given `axis_request_id`, show:

- received timestamp;
- actor context;
- SQL fingerprint;
- decision;
- matched rule;
- policy version;
- backend reached yes/no;
- bytes forwarded;
- backend command complete if any;
- audit hash chain pointer;
- approval ticket if any.

### Transaction State

- active sessions by state;
- InTransaction count;
- FailedTransaction count;
- PoisonedByAXIS count;
- safety rollback attempts;
- long-lived transaction list.

### Execution Unknown

- latest execution unknown events;
- backend dispatch intent status;
- manual reconciliation status;
- affected actor/application;
- suggested operator action.

## Health Checks

Readiness fails if:

- audit WAL cannot be appended;
- audit circuit breaker open;
- policy manifest invalid;
- backend unavailable in strict backend-required mode;
- approval store unavailable and approval policies active;
- instance policy version stale;
- disk below audit low-water mark;
- cert expired/invalid in TLS mode;
- node is draining;
- mode is hard bypass.

Liveness should not restart AXIS simply because backend is down. Restarting a healthy process because its dependency is down is how orchestration systems cosplay as chaos monkeys.

## Diagnostic Commands / Views

Future CLI/API should provide:

- `axisctl sessions list`
- `axisctl sessions show <session_id>`
- `axisctl request show <axis_request_id>`
- `axisctl audit head`
- `axisctl audit verify --since <time>`
- `axisctl approvals show <ticket>`
- `axisctl mode status`
- `axisctl drain start`
- `axisctl drain status`

## Current Known Weaknesses

- Dashboards are not part of lab Simple Query PoC unless scoped.
- On-call runbooks must be written before customer pilot.
- Metrics can leak information if label design is careless.
- OpenTelemetry integration adds overhead and must be sampled.

## Success Looks Like

An on-call engineer can diagnose a blocked or unknown query in under three minutes without SSH-ing into random containers and sacrificing dignity to grep.

## Failure Looks Like

The only observable signal is “app got 500.” That is not observability. That is a haunted house.
