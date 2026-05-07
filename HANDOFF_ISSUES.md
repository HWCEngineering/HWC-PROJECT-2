# Handoff Issues & Flags

Issues identified during handoff preparation. These should be addressed as the project transitions to company ownership.

## Security Concerns

| Issue | Location | Severity | Action |
|-------|----------|----------|--------|
| CORS allows all origins (`*`) | `apps/api/main.py` | Medium | Restrict to frontend domain in production |
| No API authentication | `apps/api/main.py` | Medium | Add auth layer before exposing publicly |
| Public blob access | `apps/api/storage/az.py` | Low | Acceptable for read-only assets; review if sensitive data added |
| `AZURE_STORAGE_CONNECTION_STRING_2` referenced in README | `apps/api/README.md` | Low | Inconsistent env var naming — code uses `AZURE_STORAGE_CONNECTION_STRING` |

## Hardcoded/Personal References (Must Update)

| Item | Location | Current Value | Action |
|------|----------|---------------|--------|
| Docker Hub image | `.github/workflows/backend.yml` | `mfalana/hwc-potree-api` | Replaced by GHCR in new workflows |
| GitHub URL in API docs | `apps/api/main.py` | `github.com/MaFalana/HWC-POTREE-API` | Update to company repo URL |
| GitHub URL in API_DOCUMENTATION.md | `API_DOCUMENTATION.md` | `github.com/MaFalana/HWC-POTREE-API` | Update to company repo URL |
| Resource group name | Old workflow | `LiDAR-Breakline-Generator` | Use variable `${{ secrets.AZURE_RESOURCE_GROUP }}` |
| Static Web App token name | Old frontend.yml | `AZURE_STATIC_WEB_APPS_API_TOKEN_GREEN_MUSHROOM_0C30E6E0F` | Renamed to generic `AZURE_STATIC_WEB_APPS_TOKEN` |
| Git remote origin | `.git/config` | `github.com/MaFalana/cloud-viewer` | Update to `github.com/MaFalana/HWC-PROJECT-2.git` |

## Stale Documentation

| File | Issue |
|------|-------|
| `API_DOCUMENTATION.md` | References `your-production-url.com` placeholder; links to personal GitHub |
| `REFACTORING_SUMMARY.md` | Historical record — consider archiving or removing |
| `FRONTEND_ORTHO_IMPLEMENTATION.md` | Implementation plan doc — move to issues/wiki after completion |
| `apps/api/README.md` | Duplicates content; references Docker Hub deployment; has duplicated sections |

## Missing Configurations

| Item | Description | Action |
|------|-------------|--------|
| `.env.example` at root | No root-level env example | Create one referencing both apps |
| Frontend `.env.example` naming | File is `env.example` not `.env.example` | Rename to `.env.example` |
| No linter config | No ESLint/Prettier for frontend, no ruff/black for backend | Add linter configs |
| No pre-commit hooks | No automated formatting on commit | Add husky + lint-staged or similar |
| Missing `.gitignore` for `apps/api/bin/` | PotreeConverter binary may be committed | Verify binary isn't in git; add to .gitignore if needed |

## Dependency Concerns

| Item | Location | Issue |
|------|----------|-------|
| Unpinned Python deps | `apps/api/requirements.txt` | No version pins on most packages — risk of breaking changes |
| React 19 (release candidate) | `package.json` | React 19 is relatively new — verify stability |
| `georaster` deps in root | `package.json` | Referenced in REFACTORING_SUMMARY but not in current package.json |
| GDAL version range | `requirements.txt` | `>=3.8.0,<4.0.0` — may conflict with system GDAL in Docker |

## Missing Health/Quality Checks

| Item | Description |
|------|-------------|
| No smoke tests | No automated post-deploy verification beyond health check |
| No frontend tests | No unit or integration tests for React components |
| No backend tests in CI | `pytest` is in requirements but no test files found in CI |
| No load testing | No performance baseline for point cloud processing |
| No uptime monitoring | No external health check monitoring configured |

## Deployment Fragility

| Issue | Description | Mitigation |
|-------|-------------|------------|
| Single-region deployment | No failover if Azure region goes down | Consider multi-region for production |
| No database backup automation | Relies on Cosmos DB built-in backup | Verify backup policy is configured |
| Worker is in-process thread | If container restarts, in-progress jobs are lost | Jobs auto-reset on startup (acceptable) |
| No rate limiting | API has no rate limits | Add rate limiting before public exposure |
| Container scales to 0 | Cold start latency when scaling from 0 replicas | Set min-replicas to 1 for production |

## Cleanup Actions

- [x] Remove Docker Hub references from workflows (replaced by GHCR)
- [x] Create standardized `dev.yml` and `production.yml` workflows
- [x] Remove personal-account-specific secret names from workflows
- [ ] Delete old `backend.yml` and `frontend.yml` after verifying new workflows
- [ ] Rename `apps/web/env.example` to `apps/web/.env.example`
- [ ] Update git remote to `https://github.com/MaFalana/HWC-PROJECT-2.git`
- [ ] Add Python linter (ruff) configuration
- [ ] Add frontend linter (eslint) configuration
- [ ] Pin Python dependency versions
- [ ] Remove duplicate "Frontend Integration Guide" section from `apps/api/README.md`
