# Handoff Issues & Recommendations

Flagged during handoff preparation. Address these as part of the transition to company ownership.

## Security Concerns

- **CORS is wide open** (`allow_origins=["*"]`) in `apps/api/main.py`. Restrict to the production SWA domain once stable.
- **No authentication** on the API. Fine for internal use, but add auth if exposed publicly.
- **Azure Blob containers use public access** (`PublicAccess.Blob`). Evaluate if SAS tokens would be more appropriate.
- **`GHCR_READ_TOKEN`** should be a company-owned PAT, not a personal one.

## Missing Items

- **No `.env.example` at root** — Created during this handoff (pointer file).
- **No linter configured** for frontend (no ESLint/Prettier config). Consider adding for consistency.
- **No Python linter configured** (no ruff/black/flake8). Consider adding `ruff` to the dev workflow.
- **No automated tests run in CI** — backend has `pytest` in requirements but no test files found in the monorepo. Add smoke tests.
- **No health check smoke test in CI** — after deployment, consider hitting `/health` to verify.
- **No `staticwebapp.config.json`** for SWA routing rules (may need for SPA fallback with `/cloud-viewer` base path).

## Stale / Removed Items

- Removed `REFACTORING_SUMMARY.md` (historical, not operational)
- Removed `FRONTEND_ORTHO_IMPLEMENTATION.md` (implementation complete)
- Removed `apps/api/.github/workflows/deploy.yml` (stale standalone repo workflow)
- Removed `apps/api/.kiro/` (development specs, not needed for production)
- Removed `apps/api/docs/DEPLOYMENT_GUIDE.md` and 6 other stale docs (Docker Hub references)
- Removed `.github/workflows/backend.yml` (Docker Hub-based, replaced by `production.yml`)
- Removed `.github/workflows/frontend.yml` (personal token references, replaced by `production.yml`)

## Hardcoded / Personal References (Fixed)

- `mfalana/hwc-potree-api` Docker Hub image — replaced with GHCR
- `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN` secrets — no longer needed
- `AZURE_STATIC_WEB_APPS_API_TOKEN_GREEN_MUSHROOM_0C30E6E0F` — replaced with generic `AZURE_STATIC_WEB_APPS_API_TOKEN`
- `LiDAR-Breakline-Generator` resource group hardcoded in old workflows — now uses `secrets.AZURE_RESOURCE_GROUP`
- GitHub references to `MaFalana/HWC-POTREE-API` — updated to `MaFalana/HWC-PROJECT-2`

## Naming Inconsistencies

- Root package is named `hwc-cloud-viewer-monorepo` but the SWA is `hwc-survey-cloud-viewer`. Align if desired.
- `AZURE_STORAGE_CONNECTION_STRING` vs `AZURE_STORAGE_CONNECTION_STRING_2` referenced in some API docs — standardized to `AZURE_STORAGE_CONNECTION_STRING`.
- `AZURE_MONGO_NAME` variable referenced in old backend workflow vs `NAME` used in code — standardized to `NAME`.

## Deployment Fragility

- Frontend build requires `PUBLIC_API_BASE_URL` at build time (Astro bakes it in). The new `production.yml` solves this with auto-discovery, but if the backend FQDN ever changes (e.g., container app recreated), a frontend redeploy is needed.
- No blue/green or canary deployment strategy. Container App revisions could be used for this.

## Outdated Dependencies (Review)

- `react: ^19.2.3` — very recent, verify compatibility with all packages
- `apps/api/requirements.txt` still exists (REFACTORING_SUMMARY said it was deleted and migrated to pyproject.toml, but it's still present). The Dockerfile uses it, so keep it but verify it's the source of truth.

## Recommendations for Company Transition

1. Transfer repository to company GitHub org
2. Create company-owned Azure service principal for `AZURE_CREDENTIALS`
3. Create company-owned PAT for `GHCR_READ_TOKEN`
4. Update `AZURE_STATIC_WEB_APPS_API_TOKEN` to point to company-owned SWA
5. Consider making the repository private
6. Add branch protection rules on `main` and `dev` (require PR, require CI pass)
7. Add CODEOWNERS file
8. Set up Dependabot for dependency updates
