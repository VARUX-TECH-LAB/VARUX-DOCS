# AXIS Enterprise Deployment Boundary

AXIS protects database operations that pass through AXIS. It cannot protect SQL sent directly to PostgreSQL with credentials or network access that bypass AXIS.

## Enforced Topology

```text
Application / Agent / Internal Tool
        |
        v
      AXIS
        |
        v
PostgreSQL private network
```

In the enterprise compose profile, AXIS is the only service with access to the private PostgreSQL network. PostgreSQL is not published on host port 5432.

```bash
docker compose -f docker-compose.enterprise.yml up --build
```

AXIS is exposed for local validation on:

```text
http://127.0.0.1:65431
```

Enterprise evidence is written to:

```text
./enterprise-data
```

Expected evidence includes `audit.log`, `audit.wal`, `approvals.sqlite` when approvals are used, and enterprise boundary reports.

## Blocked Topology

```text
Application / Agent / Internal Tool
        |
        v
PostgreSQL direct write access
```

This topology is a deployment failure for protected write paths. Production deployments must enforce private DB networking, firewall rules, security groups, IAM controls, and PostgreSQL role separation so application clients do not receive direct write-capable database credentials.

## Authenticated Request Context

When `AXIS_AUTH_MODE=jwt_hs256`, `/query` requires `Authorization: Bearer <token>`. AXIS derives actor, actor type, app, tenant, role, host, and environment from the verified token before policy evaluation and audit. Spoofed JSON body fields are ignored; conflicts are recorded in request audit payloads.

HS256 mode is for local enterprise validation only. Production identity should use OIDC/JWKS, mTLS, service identity, cloud IAM, or equivalent trusted identity infrastructure.
