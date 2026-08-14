# Source Review Mode

Architecture-level feedback requires source visibility. Black-box Docker behavior is not enough for AppSec, platform, or SRE review.

This source review package is intended only for selected trusted reviewers. It is not a public open-source release. Do not redistribute without permission.

Reviewers should focus on:

- Architecture.
- Enforcement boundaries.
- Bypass paths.
- Transaction handling.
- Audit evidence.
- Integration assumptions.
- Source/code organization.
- Policy decision flow.

Use `docs/reviewer/AXIS_CLAIMS_MATRIX.md` and `docs/reviewer/AXIS_ANTI_CLAIMS.md` as the source of truth for what is and is not claimed.
