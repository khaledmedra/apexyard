# damana-v2 — Handover Assessment

**Date**: 2026-05-27
**Assessor**: khaled medra
**Status**: handover

## Origin

- **Where it came from**: Owner's primary product — Backup-as-a-Service platform
- **Original owner**: khaledmedra
- **Repo location**: https://github.com/khaledmedra/damana-v2
- **First commit date**: 2026-02-11
- **Last commit date**: 2026-05-26

## Current State

### Tech stack

- **Language**: Python 3.12 (primary), Go 1.25 (gRPC server + agent)
- **Runtime**: Gunicorn (Django), native Go binary (gRPC server + agent)
- **Framework**: Django 6.0.1, Celery 5.6.2 + Celery Beat, gRPC (grpcio ≥1.70)
- **Database**: TimescaleDB / PostgreSQL 16 via psycopg3
- **Cache / broker**: Redis 7 (Celery broker + rate limiting + CRL store)
- **Object storage**: MinIO (S3-compatible, accessed via boto3)
- **Log store**: OpenSearch 2.19.2
- **Observability**: OpenTelemetry (traces, metrics, logs) → OTel Collector, Prometheus, Grafana
- **Frontend**: Django templates + Tailwind CSS v4 + HTMX + ApexCharts + Flowbite (no SPA)
- **Test framework**: pytest + pytest-django, Playwright (E2E), factory-boy, freezegun, moto[s3]
- **Lint / SAST**: ruff (pyproject.toml), 10 custom opengrep rules (.opengrep/)

### Build status

- `pip install -r requirements.txt`: not attempted (no local clone)
- `python manage.py check`: not attempted
- `pytest`: not attempted
- `npm run build:css`: not attempted

### Test coverage

- Estimated: **unknown** — no coverage threshold configured in pytest.ini or CI. Tests directory is present and structured (tests/auth/, tests/core/, tests/catalog/, tests/e2e/, etc.) but no coverage reporting baseline exists.

### Repo activity

- Commits in last 90 days: **100+** (very active)
- Open issues: **30**
- Open PRs: **1** (feat: add backfill_retry_copy_config management command — #258)
- Top contributors: khaled medra (primary), khaledmedra (same person via two identities)

## Harnessability assessment

**Overall verdict**: `low`

> ⚠ Harnessability: LOW
>
> Rex's architecture handbooks will fire advisory-only on this codebase. The blocking gate (`ENFORCEMENT: blocking`) will generate false positives. Recommended: adopt as advisory-only, plan a follow-up to add the missing scaffolding (mypy strict, coverage thresholds, full CI pipeline).

| Dimension | Score | Evidence |
|-----------|-------|----------|
| Type safety | `partial` | `pyproject.toml` has ruff configured but no `[tool.mypy]` strict, no `pyrightconfig.json`; Python 3.12 with no static type enforcement beyond lint |
| Module boundaries | `partial` | Clear service-level split (`core/`, `dashboard/`, `platform_admin/`, `grpc_server_go/`, `agent/`) and internal `models/` / `services/` / `tasks/` layers inside `core/`, but no formal `domain/application/infrastructure` clean-architecture structure |
| Framework opinionation | `strong` | Django 6 (full ORM, auth, admin, views, middleware), Celery (async tasks, beat), gRPC — full persistence + HTTP + async opinionation |
| Test coverage signal | `absent` | `pytest.ini` has no `--cov` flag or threshold; no coverage step in `.github/workflows/check-adapter-sync.yml`; no `.coveragerc` |
| Lint baseline | `present` | `pyproject.toml` has `[tool.ruff]` config; 10 custom opengrep SAST rules in `.opengrep/`; `.semgrepignore` present |

See AgDR-0042 for the scoring rationale and v1 thresholds.

## Quality Risks

### Security

- **Existing audit (2026-03-06)**: Full platform security audit found **15 CRITICAL, 36 HIGH, 47 MEDIUM, 31 LOW** findings. `SECURITY_AUDIT_REPORT.md` is committed at repo root. Selected CRITICAL findings include:
  - C-01: Rate limiter middleware fails open on Redis error (comment says fail-closed; code is fail-open)
  - C-02: Lockout service trusts X-Forwarded-For unconditionally
  - Plus 13 additional CRITICAL items in the report
- **Active P0 bug #252**: gRPC CRL never published — Redis key `damana:crl:current` missing; certificate revocation enforcement is completely bypassed. Tagged `domain:security` + `priority:P0`.
- CA private key material, HMAC keys, and TLS certificates managed via environment variables (`DAMANA_CA_KEY_PEM_B64`, `TOKEN_HMAC_KEY`) — high blast radius if mis-configured.
- Custom opengrep rules exist for tenant isolation and fail-secure patterns — shows security awareness but gaps remain.

### Dependencies

- No automated dependency audit in CI (single workflow only checks adapter sync)
- `cryptography==42.0.0` pinned — not latest (44.x as of audit date); may have downstream CVEs
- `Django==6.0.1` — very recent major version; ecosystem compatibility should be monitored
- `grpcio>=1.70.0` without upper bound — float risk on minor API changes

### Technical debt

- No mypy or pyright type checking; Python codebase is untyped beyond ruff
- No test coverage threshold — coverage baseline unknown; tests exist but nothing enforcing a minimum
- 30 open issues including 1 P0 (security), 4 P1 (resilience/data bugs)
- P1 #251: SSE endpoints killed every 120s — gunicorn sync workers incompatible with long-lived streams
- P1 #250: Orphan catalog entries accumulating at >1M/6h — reaper can't keep up
- P1 #249: `reap_orphan_catalog_entries` soft-timeout kills psycopg connection mid-commit

### Operational

- CI is minimal: only one workflow (`check-adapter-sync.yml`) covering a narrow adapter interface check; no automated test run, no lint gate, no security scan in CI
- OpenTelemetry + Prometheus + Grafana + OpenSearch stack is wired up — observability is a strength
- `deploy/runbooks/` directory present — runbooks exist

## Integration Plan

### Roles that apply

- `tech-lead`
- `backend-engineer` (Django, Python, Celery)
- `frontend-engineer` (Django templates, Tailwind v4, HTMX)
- `platform-engineer` (CI pipeline, deploy scripts, systemd units)
- `sre` (OTel stack, Prometheus/Grafana/OpenSearch, deploy/runbooks/)
- `security-auditor` (auth middleware, crypto/CA infrastructure, gRPC CRL, multi-tenant isolation)

### Workflows that kick in

- [ ] PR workflow (`.claude/rules/pr-workflow.md`) — every change goes through a PR
- [ ] AgDR for technical decisions
- [ ] Code Reviewer agent on every PR
- [ ] Security Reviewer agent on first pass and all auth/crypto/gRPC PRs (large attack surface)
- [ ] `/audit-deps` on adoption and monthly thereafter

### Hooks to enable (in damana-v2's own repo)

- [ ] `block-git-add-all`
- [ ] `block-main-push`
- [ ] `validate-branch-name`
- [ ] `validate-pr-create`
- [ ] `pre-push-gate`
- [ ] `check-secrets` (CA keys + HMAC keys in env vars — critical)

### CI templates to copy in (from ApexYard golden-paths)

- [ ] `golden-paths/pipelines/ci.yml` — replaces the single-workflow setup
- [ ] `golden-paths/pipelines/security.yml` — opengrep/semgrep already partially in place
- [ ] `golden-paths/pipelines/pr-title-check.yml`
- [ ] `golden-paths/pipelines/dependency-audit.yml`

### Registry entry

```yaml
- name: damana-v2
  repo: khaledmedra/damana-v2
  workspace: workspace/damana-v2
  docs: projects/damana-v2
  status: handover
  roles:
    - tech-lead
    - backend-engineer
    - frontend-engineer
    - platform-engineer
    - sre
    - security-auditor
```

## Next Steps

1. ~~Address the 15 CRITICAL + 36 HIGH findings from `SECURITY_AUDIT_REPORT.md` (2026-03-06) — triage into P0/P1 tickets before any new feature work; revocation bypass (#252) is the most urgent~~ → Filed as [#261](https://github.com/khaledmedra/damana-v2/issues/261)
2. Fix P0 bug #252 — gRPC CRL never published; Redis key `damana:crl:current` missing means certificate revocation is not enforced across the fleet (already tracked as [#252](https://github.com/khaledmedra/damana-v2/issues/252))
3. ~~Set up full CI pipeline — copy `golden-paths/pipelines/ci.yml` + `security.yml` + `dependency-audit.yml`; the current single workflow covers only adapter sync~~ → Filed as [#262](https://github.com/khaledmedra/damana-v2/issues/262)
4. ~~Set up test coverage reporting — add `pytest-cov` threshold (suggest `--cov-fail-under=70` as a baseline) so the PR gate catches regressions~~ → Filed as [#263](https://github.com/khaledmedra/damana-v2/issues/263)
5. ~~`/code-review` the most-recent PR on this repo as Rex to calibrate review standards~~ → Filed as [#264](https://github.com/khaledmedra/damana-v2/issues/264)
6. ~~Stakeholder sync with the previous owner to cover context the static read couldn't surface~~ → Filed as [#265](https://github.com/khaledmedra/damana-v2/issues/265)

## Post-Handover Checklist

- [ ] Review this assessment with the previous owner / team
- [ ] Triage the 15 CRITICAL findings from `SECURITY_AUDIT_REPORT.md` — close the P0 revocation bypass before the first feature PR
- [ ] Address P1 bugs #249, #250, #251 — resilience + data integrity issues scheduled in the first 2 weeks
- [ ] Add `damana-v2` to the weekly `/stakeholder-update` rollup
- [ ] Onboard tech-lead, backend-engineer, sre, security-auditor into the review rotation
- [ ] Set up test coverage baseline (`pytest --cov --cov-report=term`) and commit a threshold
- [ ] Run `/audit-deps damana-v2` monthly for the next 3 months

## Open Questions

- What is the current deployment environment (self-hosted, cloud provider, customer-premise)?
- Are the 15 CRITICAL findings from the March 2026 audit tracked as individual GitHub issues, or only in the audit report?
- What is the intended SLA / RTO for the backup service?
- Is there a staging environment separate from dev?
- What's the current customer count / data volume (relevant to the P1 orphan-catalog accumulation rate)?
