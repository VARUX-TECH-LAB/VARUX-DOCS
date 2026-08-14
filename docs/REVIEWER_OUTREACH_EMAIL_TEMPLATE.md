# Reviewer Outreach Email Template

## Version A: Senior Reviewer / CISO / Architect Short Email

Subject: Private technical teardown request for AXIS

Hi [NAME],

PERSONAL_REASON_FOR_THIS_REVIEWER

I am asking a small number of trusted reviewers to tear down AXIS privately before any CISO-level or public positioning.

AXIS is a deterministic control layer for protected PostgreSQL write paths. The current package shows a FastAPI + SQLAlchemy pilot where protected ORM writes go through policy enforcement, approval routing, rollback-safe retry, and audit evidence.

Demo video: VIDEO_LINK_PLACEHOLDER  
Hosted overview: HOSTED_OVERVIEW_LINK_PLACEHOLDER  
Package download: PACKAGE_DOWNLOAD_LINK_PLACEHOLDER

Could you give me a 15-minute live teardown? Written form is optional. Harsh feedback is preferred, especially on architecture, bypass paths, deployment realism, and whether this is worth piloting.

This is not a sales pitch in disguise. AXIS is in the engineering validation phase, not GTM. I am asking for architectural tear-down, not budget.

Turkish version: Bu gizli bir satış mesajı değildir. AXIS şu anda GTM/satış aşamasında değil, mühendislik doğrulama aşamasındadır. Bütçe değil, mimariyi acımasızca eleştirmenizi istiyorum.

## Version B: Technical Reviewer Email

Subject: Source-visible AXIS reviewer package - 15-minute teardown request

Hi [NAME],

PERSONAL_REASON_FOR_THIS_REVIEWER

AXIS is a deterministic control layer for protected PostgreSQL write paths.

This private reviewer package demonstrates AXIS protecting ORM-generated write operations in a realistic FastAPI + SQLAlchemy backend using policy enforcement, approval routing, rollback-safe retry, and audit evidence.

What is inside:
- Source-visible technical review package.
- Docker Compose pilot stack.
- Smoke tests and demo script.
- Evidence capture and verification.
- Pre-generated evidence for no-run review.
- Diagnostics collector if setup fails.
- Claims, anti-claims, bypass boundary, and attack checklist docs.

What is not claimed:
- Native PostgreSQL wire compatibility.
- Transparent enterprise drop-in proxy support.
- Universal ORM coverage.
- Production deployment readiness.
- Enterprise-scale benchmark results.

Start here: `demo/REVIEWER_START_HERE.md`

If setup fails:
- Windows: `python scripts\collect_reviewer_diagnostics.py`
- macOS/Linux: `python3 scripts/collect_reviewer_diagnostics.py`

I am asking for a 15-minute live teardown. Written form is optional. The feedback areas I care about most are architecture credibility, bypass paths, transaction and approval semantics, audit evidence, deployment realism, and whether this is pilot-worthy with current limitations.

This is not a sales pitch in disguise. AXIS is in the engineering validation phase, not GTM. I am asking for architectural tear-down, not budget.

Turkish version: Bu gizli bir satış mesajı değildir. AXIS şu anda GTM/satış aşamasında değil, mühendislik doğrulama aşamasındadır. Bütçe değil, mimariyi acımasızca eleştirmenizi istiyorum.
