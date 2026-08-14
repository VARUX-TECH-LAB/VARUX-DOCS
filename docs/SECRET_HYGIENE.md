# AXIS Secret Hygiene

AXIS Secret Hygiene & Key Identity Hardening keeps secrets out of routine runtime output and fails fast when production profile configuration uses local, demo, default, empty, or too-short credentials.

## Redaction Scope

Runtime error details, runtime log metadata, and evidence export metadata are passed through centralized redaction before they are returned or stored for operator review.

AXIS redacts these patterns at minimum:

- `Authorization: Bearer <token>` and standalone bearer tokens
- JWT-like compact tokens
- `AXIS_OPERATOR_TOKEN`
- `AXIS_JWT_HS256_SECRET`
- `password=...` connection-string fields
- `postgres://user:password@host/db`
- `secret=...`
- `token=...`
- `signing_key=...`
- `private_key=...`
- private-key PEM looking material

The replacement marker is:

```text
<REDACTED>
```

AXIS still returns useful diagnostics such as error codes, categories, safe reason codes, policy ids, request ids, limits, and audit verification state. It should not return raw credentials, bearer tokens, signing private keys, database URLs with passwords, raw authorization headers, or local `.env` contents.

## Weak Secret Rules

Production profile rejects weak/default secrets for operator tokens and JWT HS256 secrets.

Rejected examples include:

- empty values
- too-short values
- `change-me`
- `demo`
- `demo-secret`
- `test`
- `test-secret`
- `secret`
- `password`
- `123456`
- local/demo-labeled tokens such as `axis-enterprise-local-operator-token-000000000000`

When `AXIS_AUTH_MODE=jwt_hs256`, `AXIS_JWT_HS256_SECRET` is required. In `AXIS_RUNTIME_PROFILE=production`, it must be a strong non-default value.

`AXIS_RUNTIME_PROFILE=local` may use clearly labeled local-only demo secrets so reviewer and enterprise boundary validation remain runnable. Those values are not production credentials and are rejected if production profile is enabled.

## Production Expectations

Production deployments should set:

```text
AXIS_RUNTIME_PROFILE=production
AXIS_OPERATOR_TOKEN=<strong managed operator token>
```

If JWT HS256 is temporarily used:

```text
AXIS_AUTH_MODE=jwt_hs256
AXIS_JWT_REQUIRED=true
AXIS_JWT_HS256_SECRET=<strong managed local validation secret>
```

HS256 remains a local/demo authentication mode. Production identity should move to OIDC/JWKS, mTLS, cloud IAM, service identity, or equivalent trusted identity infrastructure.

## What Not To Commit

Never commit:

- `.env` files containing real values
- operator tokens
- JWT signing secrets
- PostgreSQL passwords or full database URLs with passwords
- evidence signing private keys
- KMS, Vault, or cloud secret-store credentials
- runtime audit/evidence data directories
- core dumps, crash dumps, or debug captures containing live memory

The repository may contain local demo placeholders and example values. They are intentionally labeled as local/demo and are not production credentials.

## Supplying Secrets

Provide secrets through deployment tooling, environment injection, or a managed secret store. Do not place real values in source files, Docker images, frontend environment variables, screenshots, tickets, or review reports.

Operator-protected endpoints accept `X-AXIS-Operator-Token` or `Authorization: Bearer <token>`. Browser code should not receive operator tokens; the control plane proxy is expected to inject server-side credentials.

## Validation

Run the Rust tests that cover redaction, weak-secret validation, operator auth errors, health output, and evidence signing metadata:

```bash
cargo test
```

For runtime validation, check that `/health`, `/runtime/metrics`, `/logs`, auth failures, and evidence exports do not include bearer tokens, `AXIS_OPERATOR_TOKEN`, `AXIS_JWT_HS256_SECRET`, database passwords, private keys, or full unredacted connection strings.
