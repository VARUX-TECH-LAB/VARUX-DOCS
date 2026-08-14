# DB Access Hardening

AXIS runtime must not use PostgreSQL superuser credentials. Enterprise deployments should separate migration, AXIS execution, and application read-only access.

## Roles

`migration_admin`

Owns the enterprise demo schema and is used only for schema setup and migrations. This role is not used by AXIS runtime.

`axis_executor`

Runtime role used by AXIS. It is `NOSUPERUSER`, does not own `prod_main`, has schema `USAGE`, and has scoped DML privileges on the enterprise demo tables. It does not receive broad DDL privileges.

`app_readonly`

Read-only proof role. It can `SELECT` permitted demo tables and cannot `INSERT`, `UPDATE`, `DELETE`, `DROP`, `ALTER`, or `TRUNCATE`. This role is used by `scripts/enterprise_boundary_check.py` to prove direct app write bypass fails.

## Apply Roles

The enterprise compose profile applies:

```text
sql/enterprise_roles.sql
sql/enterprise_seed.sql
```

on first PostgreSQL volume initialization.

For an existing database, review the scripts first, then apply through a controlled migration path as a bootstrap PostgreSQL administrator. Do not run AXIS as that administrator.

## Verify Grants

Run:

```bash
python scripts/enterprise_boundary_check.py
```

The check verifies:

- PostgreSQL is not exposed on host port 5432
- `app_readonly` cannot write directly
- `axis_executor` is not superuser
- `axis_executor` does not own `prod_main`

Manual checks:

```sql
SELECT rolname, rolsuper, rolcreatedb, rolcreaterole
FROM pg_roles
WHERE rolname IN ('axis_executor', 'app_readonly', 'migration_admin');

SELECT grantee, privilege_type
FROM information_schema.role_table_grants
WHERE grantee IN ('axis_executor', 'app_readonly')
ORDER BY grantee, table_name, privilege_type;
```

## Credential Rotation

The compose file uses demo passwords only. Production deployments should source credentials from a secret manager, rotate `axis_executor` and application credentials independently, and revoke any direct write-capable application credentials before routing protected writes through AXIS.
