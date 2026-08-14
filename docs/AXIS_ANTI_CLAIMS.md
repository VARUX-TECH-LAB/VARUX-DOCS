# AXIS Anti-Claims

This package is intentionally explicit about what it does not prove.

- This package does not claim native PostgreSQL wire compatibility.
- This package does not claim transparent drop-in proxy support.
- This package does not claim universal ORM support.
- This package does not claim production deployment readiness.
- This package does not claim application-layer routing alone prevents all direct DB bypass.
- This package does not claim live human approval while holding open database transactions.
- This package does not claim offline/disconnected corporate environment support.
- This package does not claim final compliance certification.
- This package does not claim enterprise-scale performance benchmarking.
- This package does not claim public open-source availability.

The current pilot demonstrates a source-visible integration pattern: FastAPI + SQLAlchemy routes protected writes through AXIS HTTP `/query`, while safe reads may go directly to PostgreSQL. That is useful evidence for architecture review, but it is not the same as a network-level PostgreSQL wire proxy.

If a reviewer sees a stronger claim implied by wording elsewhere in the package, treat this file as authoritative and flag the mismatch.
