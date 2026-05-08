# HWC Cloud Viewer

A monorepo for the HWC Survey Cloud Viewer platform — a web application for managing, processing, and visualizing LiDAR point cloud data and orthophotos.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        GitHub (CI/CD)                            │
│  dev.yml: lint/build validation                                 │
│  production.yml: deploy API → capture URL → build & deploy SWA  │
└────────────┬──────────────────────────────────┬─────────────────┘
             │                                  │
             ▼                                  ▼
┌────────────────────────┐       ┌──────────────────────────────┐
│  Azure Container Apps  │       │  Azure Static Web Apps        │
│  (hwc-potree-api)      │       │  (hwc-survey-cloud-viewer)    │
│  FastAPI + Python 3.12 │       │  Astro + React                │
│  Port 8000             │◄──────│  PUBLIC_API_BASE_URL (build)  │
└────────────┬───────────┘       └──────────────────────────────┘
             │
     ┌───────┴───────┐
     ▼               ▼
┌──────────┐  ┌──────────────┐
│ Cosmos DB│  │ Azure Blob   │
│ (Mongo)  │  │ Storage      │
└──────────┘  └──────────────┘
```

## Repository Structure

```
HWC-PROJECT-2/
├── .github/workflows/
│   ├── dev.yml              # CI: lint, build, validate on dev/PRs
│   └── production.yml       # CD: deploy API + frontend on main
├── apps/
│   ├── api/                 # FastAPI backend (Python 3.12)
│   │   ├── config/          # App configuration and initialization
│   │   ├── models/          # Pydantic data models
│   │   ├── routes/          # API route handlers
│   │   ├── storage/         # Azure Blob + MongoDB managers
│   │   ├── bin/             # PotreeConverter binary
│   │   ├── Dockerfile       # Container image definition
│   │   └── main.py          # FastAPI app entry point
│   └── web/                 # Astro + React frontend
│       ├── src/
│       │   ├── api/         # API client classes
│       │   ├── components/  # React components (Dashboard, Viewer)
│       │   ├── pages/       # Astro pages
│       │   └── styles/      # Global CSS
│       └── astro.config.mjs
├── packages/                # Shared internal packages (npm workspaces)
│   ├── assets/              # Static branding assets
│   ├── header/              # Header component
│   ├── map/                 # 2D Leaflet map component
│   ├── panel/               # Collapsible panel component
│   ├── potree/              # Potree 1.8.2 viewer files
│   └── ui/                  # Reusable UI primitives
├── scripts/
│   ├── sync-assets.mjs      # Copy assets to app public dirs
│   └── sync-potree.mjs      # Copy Potree files to web public dir
└── package.json             # Root workspace config
```

## Tech Stack

| Layer      | Technology                                    |
|------------|-----------------------------------------------|
| Frontend   | Astro 5, React 19, Leaflet, Potree 1.8.2     |
| Backend    | FastAPI, Python 3.12, Uvicorn                 |
| Database   | Azure Cosmos DB (MongoDB API)                 |
| Storage    | Azure Blob Storage (public containers)        |
| Hosting    | Azure Static Web Apps (frontend)              |
| Compute    | Azure Container Apps (backend)                |
| Registry   | GitHub Container Registry (ghcr.io)           |
| CI/CD      | GitHub Actions                                |
| Processing | PotreeConverter, GDAL                         |

## Local Development Setup

### Prerequisites

- Node.js 20+
- Python 3.12+
- Docker (optional, for API container testing)
- Azure CLI (for deployment testing)

### Frontend

```bash
npm install
npm run dev:web
```

### Backend

```bash
cd apps/api
pip install -r requirements.txt
# Copy .env.example to .env and fill in values
cp .env.example .env
python -m uvicorn main:app --reload --port 8000
```

### Full Stack (concurrent)

```bash
npm run dev
```

## Environment Variables / Secrets

### Backend (apps/api/.env)

| Variable                         | Description                              | Required |
|----------------------------------|------------------------------------------|----------|
| `MONGO_CONNECTION_STRING`        | Cosmos DB (Mongo API) connection string   | Yes      |
| `AZURE_STORAGE_CONNECTION_STRING`| Azure Blob Storage connection string      | Yes      |
| `NAME`                           | Database/container name (e.g. project ID) | Yes      |
| `POTREE_PATH`                    | Path to PotreeConverter binary            | Yes (Docker sets this) |
| `PORT`                           | Server port (default: 8000)              | No       |

### Frontend (apps/web/.env)

| Variable                 | Description                          | Required |
|--------------------------|--------------------------------------|----------|
| `PUBLIC_API_BASE_URL`    | Backend API URL                      | Yes      |
| `PUBLIC_MAPTILER_API_KEY`| MapTiler API key for map tiles       | Yes      |
| `PUBLIC_APP_BASE_PATH`   | Base path for the app (default: /cloud-viewer) | No |

### GitHub Secrets (Repository Settings → Secrets)

| Secret                            | Description                                      |
|-----------------------------------|--------------------------------------------------|
| `AZURE_CREDENTIALS`              | Azure service principal JSON for CLI login         |
| `AZURE_RESOURCE_GROUP`           | Azure resource group name                          |
| `AZURE_STATIC_WEB_APPS_API_TOKEN`| Deployment token for the Static Web App           |
| `MONGO_CONNECTION_STRING`        | Production Cosmos DB connection string             |
| `AZURE_STORAGE_CONNECTION_STRING`| Production Azure Blob Storage connection string    |
| `GHCR_READ_TOKEN`               | PAT with `read:packages` scope for container pull  |

### GitHub Variables (Repository Settings → Variables)

| Variable                  | Description                              |
|---------------------------|------------------------------------------|
| `NAME`                    | Database/container name                  |
| `PUBLIC_MAPTILER_API_KEY` | MapTiler API key                         |

## Build & Run Commands

| Command              | Description                              |
|----------------------|------------------------------------------|
| `npm install`        | Install all workspace dependencies       |
| `npm run dev`        | Run frontend + backend concurrently      |
| `npm run dev:web`    | Run frontend dev server only             |
| `npm run dev:api`    | Run backend dev server only              |
| `npm run build`      | Build frontend for production            |
| `npm run preview`    | Preview production build locally         |
| `npm run assets:sync`| Sync shared assets to app public dirs    |
| `npm run potree:sync`| Sync Potree files to web public dir      |

## CI/CD Overview

### Branching Strategy

```
feature/* → dev → main
```

- `feature/*` branches: individual work
- `dev`: integration branch, CI runs on every push/PR
- `main`: production branch, triggers deployment

### Workflow: dev.yml

Triggers on push to `dev` or PRs targeting `dev`. Runs:
- Frontend: install → sync assets → build (validates compilation)
- Backend: syntax check → Docker image build (validates Dockerfile)

No deployment occurs. This is validation only.

### Workflow: production.yml

Triggers on push to `main`. Two sequential jobs:

1. `deploy-api`: Build container → push to GHCR → deploy to Azure Container Apps → capture FQDN
2. `deploy-frontend`: Install → build with discovered API URL → deploy to Azure Static Web Apps

The API URL auto-discovery eliminates the need to manually update `PUBLIC_API_BASE_URL` after backend deployment.

## Infrastructure Setup (From Scratch)

### Required Azure Resources

1. **Resource Group** — logical container for all resources
2. **Azure Cosmos DB account** (MongoDB API) — project/job data
3. **Azure Storage Account** — blob storage for point clouds, orthophotos, thumbnails
4. **Azure Container App Environment** — hosting environment for the API container
5. **Azure Container App** (`hwc-potree-api`) — the backend API
6. **Azure Static Web App** (`hwc-survey-cloud-viewer`) — the frontend

### Setup Steps

```bash
# 1. Create resource group
az group create --name <RESOURCE_GROUP> --location eastus2

# 2. Create Cosmos DB (MongoDB API)
az cosmosdb create --name <COSMOS_ACCOUNT> --resource-group <RESOURCE_GROUP> --kind MongoDB
az cosmosdb mongodb database create --account-name <COSMOS_ACCOUNT> --resource-group <RESOURCE_GROUP> --name <DB_NAME>

# 3. Create Storage Account
az storage account create --name <STORAGE_ACCOUNT> --resource-group <RESOURCE_GROUP> --sku Standard_LRS
# Get connection string from portal or CLI

# 4. Create Container App Environment
az containerapp env create --name LiDAR-CONTAINER --resource-group <RESOURCE_GROUP> --location eastus2

# 5. Create Static Web App
az staticwebapp create --name hwc-survey-cloud-viewer --resource-group <RESOURCE_GROUP> --location eastus2
# Retrieve the deployment token from the portal

# 6. Create Azure Service Principal for GitHub Actions
az ad sp create-for-rbac --name "github-actions-hwc" --role contributor --scopes /subscriptions/<SUB_ID>/resourceGroups/<RESOURCE_GROUP> --sdk-auth
# Save the JSON output as AZURE_CREDENTIALS secret
```

### GHCR Setup

1. Create a Personal Access Token (classic) with `read:packages`, `write:packages` scopes
2. Store as `GHCR_READ_TOKEN` in repository secrets
3. The `GITHUB_TOKEN` (automatic) handles push during CI; `GHCR_READ_TOKEN` is used by Azure to pull images

### Environment Separation

| Concern       | Dev                        | Production                          |
|---------------|----------------------------|-------------------------------------|
| Branch        | `dev`                      | `main`                              |
| CI            | Lint + build validation    | Full deploy                         |
| API hosting   | Local (localhost:8000)     | Azure Container Apps                |
| Frontend      | Local Astro dev server     | Azure Static Web Apps               |
| Database      | Local/dev Cosmos instance  | Production Cosmos instance          |
| Registry      | N/A (no push)             | ghcr.io                             |

## Deployment

### How Backend/Frontend Deployments Interact

The frontend requires `PUBLIC_API_BASE_URL` at build time (Astro bakes env vars into static output). The production workflow solves this by:

1. Deploying the backend first
2. Using `az containerapp show` to retrieve the live FQDN
3. Passing it as a job output to the frontend build step
4. Building the frontend with the real API URL
5. Deploying the static output

No manual variable updates or redeployments needed.

### Rollback

**Backend:**
```bash
# Find previous image tag (commit SHA)
# Redeploy with previous image
az containerapp update \
  --name hwc-potree-api \
  --resource-group <RESOURCE_GROUP> \
  --image ghcr.io/<owner>/hwc-potree-api:<previous-sha>
```

**Frontend:**
```bash
# Re-run the production workflow from a previous commit on main
# Or revert the commit and push to main
git revert <commit> && git push origin main
```

## Troubleshooting

| Problem                                | Solution                                                        |
|----------------------------------------|-----------------------------------------------------------------|
| Frontend shows "Network error"         | Check `PUBLIC_API_BASE_URL` is correct; verify API is running   |
| Container App not starting             | Check `az containerapp logs` for startup errors                 |
| Health check failing                   | Verify Cosmos DB and Blob Storage connection strings            |
| GHCR pull fails in Azure               | Verify `GHCR_READ_TOKEN` has `read:packages` scope             |
| SWA deploy fails                       | Verify `AZURE_STATIC_WEB_APPS_API_TOKEN` is correct            |
| Build fails on Potree sync             | Ensure `packages/potree/1.8.2/` directory exists                |
| CORS errors in browser                 | API allows all origins; check browser network tab for real error|
| Job stuck in "processing"              | Check API logs; restart container app to reset stale jobs       |

## Contributing

### Branching Workflow

1. Create a feature branch from `dev`: `feature/my-feature`
2. Make changes, commit with conventional commits
3. Open a PR targeting `dev`
4. CI must pass before merge
5. After testing on `dev`, open a PR from `dev` → `main` for production release

### Commit Naming Convention

```
type(scope): description

feat(api): add orthophoto upload endpoint
fix(web): correct map bounds calculation
chore(ci): update Node.js version in workflow
docs: update README with deployment steps
```

Types: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`, `style`

### Pull Request Expectations

- CI must pass (lint, build)
- Describe what changed and why
- Include testing steps if applicable
- No hardcoded secrets or personal URLs

### Formatting / Linting

- Frontend: follow existing Astro/React conventions
- Backend: PEP 8 style (consider adding `ruff` or `black` in future)
- No trailing whitespace, consistent indentation

### Before Merging to Main

- All CI checks pass on `dev`
- Manual smoke test of affected features
- No console errors in browser
- API health check returns healthy
