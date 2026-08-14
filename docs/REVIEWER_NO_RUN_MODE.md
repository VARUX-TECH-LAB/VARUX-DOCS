# Reviewer No-Run Mode

Use no-run mode if you cannot or do not want to run untrusted code locally.

## What You Can Evaluate Without Running

- Architecture.
- Evidence shape.
- Claims and anti-claims.
- Source structure, if you received the source-visible package.
- Integration boundary.
- Risk model.
- Pre-generated evidence format.
- Bypass and deployment assumptions.

## What You Cannot Evaluate Without Running

- Local reproducibility.
- Local Docker compatibility.
- Local performance.
- Actual environment-specific behavior.
- Whether your own machine can build or run the stack.

No-run review can evaluate architecture and evidence shape, but it cannot independently verify local execution.

## Where To Inspect

- Start with `demo/pre-generated-evidence/README.md`.
- Inspect selected response examples under `demo/pre-generated-evidence/pilot-v1/selected-responses/`.
- Inspect selected audit samples under `demo/pre-generated-evidence/pilot-v1/selected-audit-events/`.
- Inspect verification examples under `demo/pre-generated-evidence/pilot-v1/verification-output/`.
- Read `docs/reviewer/AXIS_CLAIMS_MATRIX.md`.
- Read `docs/reviewer/AXIS_ANTI_CLAIMS.md`.
- Read `docs/reviewer/BYPASS_BOUNDARY_AND_DEPLOYMENT_ASSUMPTIONS.md`.
- Read `docs/reviewer/SOURCE_REVIEW_MODE.md` if reviewing source.

## Feedback

Preferred feedback is a 15-minute live walkthrough or teardown. Written feedback can use `docs/reviewer/REVIEWER_FEEDBACK_FORM.md`.
