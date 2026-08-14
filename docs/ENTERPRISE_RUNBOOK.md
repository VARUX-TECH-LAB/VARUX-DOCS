# Enterprise Runbook

## Start Enterprise Profile

```bash
docker compose -f docker-compose.enterprise.yml up --build
```

AXIS listens on:

```text
http://127.0.0.1:65431
```

PostgreSQL is internal-only and must not be published on host port 5432.

The Compose profile sets `AXIS_RUNTIME_PROFILE=local` and uses explicitly labeled local/demo secrets so boundary validation can run on a reviewer workstation. Do not reuse those values with `AXIS_RUNTIME_PROFILE=production`; production profile rejects local/demo/default secrets.

## Run Boundary Check

```bash
python scripts/enterprise_boundary_check.py
```

Expected success:

```text
AXIS ENTERPRISE BOUNDARY CHECK: PASS
```

Evidence path:

```text
./enterprise-data
```

Generated reports:

```text
./enterprise-data/enterprise-boundary-report.json
./enterprise-data/enterprise-boundary-report.md
```

## Troubleshooting

AXIS not healthy

Check `docker compose -f docker-compose.enterprise.yml ps`, AXIS logs, policy manifest validation, and PostgreSQL health.

PostgreSQL exposed on 5432

Remove `ports: 5432:5432` from the enterprise PostgreSQL service and keep it only on `axis_private_db`.

`app_readonly` can write

Reapply `sql/enterprise_roles.sql`, verify table ownership, and ensure no broad grants or role memberships give write privileges.

Missing or invalid JWT accepted

Verify `AXIS_AUTH_MODE=jwt_hs256`, `AXIS_JWT_REQUIRED=true`, and `AXIS_JWT_HS256_SECRET` match the boundary check environment.

Production JWT/auth validation

Set `AXIS_RUNTIME_PROFILE=production` only with strong non-default `AXIS_OPERATOR_TOKEN` and, if HS256 is temporarily enabled, a strong non-default `AXIS_JWT_HS256_SECRET`. For production IAM, use the strategy in `docs/technical/KMS_OIDC_STRATEGY.md` instead of treating HS256 as enterprise identity.

Spoofed actor accepted

Confirm AXIS derives actor, tenant, role, app, host, actor type, and environment from the verified token before building `RequestContext`.

Audit evidence missing

Check the `./enterprise-data:/app/data` mount, file ownership, disk space, and `AXIS_AUDIT_WAL_PATH` / `AXIS_AUDIT_LOG_PATH`.

Policy manifest invalid

Run the existing policy manifest update flow only after intentional policy review. Do not bypass manifest validation to start AXIS.
