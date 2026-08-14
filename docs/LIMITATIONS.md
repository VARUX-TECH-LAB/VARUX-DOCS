# AXIS Limitations

## Review/demo scope

AXIS reviewer demo is intended for technical evaluation and pilot-readiness validation. It is not a full production deployment guide.

## PostgreSQL scope

AXIS is currently focused on PostgreSQL-oriented database control paths.

## Native wire protocol scope

If the AXIS reviewer demo uses HTTP gate mode, HTTP demo mode and native PostgreSQL wire protocol support are separate concerns. Passing the reviewer HTTP demo does not prove native PostgreSQL wire protocol coverage.

## Direct database bypass

AXIS cannot protect database traffic that bypasses AXIS entirely. Production deployments must enforce network, IAM, firewall, security group, and database access controls so protected write paths must pass through AXIS.

The enterprise compose profile demonstrates this boundary by keeping PostgreSQL off host port 5432 and by using a read-only proof role, but customer production environments must enforce the same boundary with their own network and credential controls.

## Identity boundary

JWT HS256 enterprise demo mode is not a complete enterprise IAM solution. It proves that AXIS can derive policy and audit context from a verified token instead of trusting spoofable request JSON fields. Production deployments should integrate with OIDC/JWKS, mTLS, cloud IAM, service identity, or equivalent trusted identity infrastructure.

HS256 mode remains local/demo authentication and is not full enterprise IAM. Local secrets reduce accidental leak risk for validation runs only; they do not provide managed production identity, rotation, revocation, or centralized access audit.

## Local manifest integrity

SHA-256 manifest validation proves local file integrity. It is not the same as asymmetric signing, remote attestation, or hardware-backed trust.

## Key management

Local signing and authentication secrets are not equivalent to managed KMS or HSM-backed key management. Production secret material should be generated, stored, rotated, and audited through managed key infrastructure.

AXIS does not yet provide native KMS, HSM, Vault, or cloud key store integration. Evidence signing metadata reports whether signing is disabled or signed, and which key id was used when signing is configured, but key lifecycle remains an operator responsibility.

Memory zeroing reduces accidental retention after a secret wrapper is dropped. It does not make live process memory impossible to inspect. Production deployments should disable core dumps and use managed secret infrastructure.

## High availability

A single-node AXIS deployment is not a full high availability architecture. Production designs need explicit availability, failover, observability, and recovery planning for AXIS, PostgreSQL connectivity, audit storage, and policy distribution.

## Audit durability assumptions

AXIS can generate durable audit evidence, but real production durability also depends on:

- disk reliability
- volume configuration
- backup strategy
- retention policy
- monitoring
- access controls around evidence storage

## ORM-generated SQL

AXIS uses deterministic SQL parsing and policy evaluation. ORMs such as Prisma, Hibernate, Entity Framework, Sequelize, and similar tools can generate deeply nested, complex, vendor-specific, or unusual SQL shapes.

Early pilot deployments are optimized for explicit, reviewable, and predictable SQL patterns.

Extremely complex ORM-generated query shapes may be rejected or fail-closed until they are profiled, classified, and added to the supported policy and regression corpus.

This behavior is intentional. AXIS prefers fail-closed behavior over silently allowing SQL it cannot classify with confidence.

## Performance benchmark limitation

Reviewer demo benchmark results are not a substitute for customer-specific workload testing.

## Pilot deployment boundary

Production pilot deployments should start with a narrow protected write path, defined policy scope, known database schema, and agreed rollback/incident procedure.
