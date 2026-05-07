# HWC Cloud Viewer

A monorepo for the HWC Cloud Viewer platform — a web application for visualizing LiDAR point cloud data and orthophotos. The system processes LAS/LAZ files into Potree format for 3D web visualization and converts GeoTIFF orthophotos into Cloud Optimized GeoTIFFs for 2D map overlays.

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          GitHub Repository                                │
│                  (monorepo: apps + shared packages)                       │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
              ┌──────────────┴──────────────┐
              │ GitHub Actions CI/CD         │
              │ (dev.yml / production.yml)   │
              └──────┬──────────────┬───────┘
                     │              │
         ┌───────────▼───┐   ┌─────▼────────────────┐
         │ Azure Static  │   │ Azure Container App   │
         │ Web Apps      │   │ (FastAPI backend)     │
         │ (Astro front) │   │ Image from GHCR       │
         └───────────────┘   └──────┬────────────────┘
                                    │
                     ┌──────────────┼──────────────┐
                     │              │              │
               ┌─────▼─────┐ ┌─────▼─────┐ ┌─────▼──────────┐
               │ Azure Blob │ │  MongoDB  │ │ PotreeConverter │
               │  Storage   │ │(Cosmos DB)│ │   (in-container)│
               └────────────┘ └───────────┘ └────────────────┘
```

## Repository Structure

```
hwc-cloud-viewer-monorepo/
├── .github/workflows/       # CI/CD pipelines
│   ├── dev.yml              # Dev branch: lint, build, test
│   └── production.yml       # Main branch: build, push, deploy
├── apps/
│   ├── api/                 # FastAPI backend (Python 3.12)
│   │   ├── config/          # App configuration
│   │   ├── models/          # Pydantic/data models
│   │   ├── routes/          # API route handlers
│   │   ├── storage/         # Azure Blob + MongoDB managers
│   │   ├── utils/           # Processing utilities
│   │   ├── bin/             # PotreeConverter binary (Linux)
│   │   ├── Dockerfile       # Container image definition
│   │   └── main.py          # FastAPI app entrypoint
│   └── web/                 # Astro + React frontend
│       ├── src/
│       │   ├── api/         # API client layer
│       │   ├── components/  # React components
│       │   ├── pages/       # Astro pages
│       │   └── styles/      # Global CSS
│       └── astro.config.mjs
├── packages/                # Shared workspace packages
│   ├── assets/              # Branding images
│   ├── header/              # Header component
│   ├── map/                 # Leaflet map component
│   ├── panel/               # Collapsible panel
│   ├── potree/              # Potree 1.8.2 viewer + wrapper
│   └── ui/                  # Reusable UI primitives
├── scripts/                 # Build-time sync scripts
│   ├── sync-assets.mjs      # Copy assets to app public dirs
│   └── sync-potree.mjs      # Copy Potree libs to web public
└── package.json             # Root workspace config
```

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend framework | Astro 5 + React 19 |
| Frontend hosting | Azure Static Web Apps |
| Backend framework | FastAPI (Python 3.12) |
| Backend hosting | Azure Container Apps |
| Container registry | GitHub Container Registry (ghcr.io) |
| Database | MongoDB (Azure Cosmos DB for MongoDB API) |
| File storage | Azure Blob Storage (public read) |
| Point cloud processing | PotreeConverter, laspy, pyproj |
| Geospatial processing | GDAL (COG conversion) |
| Maps | Leaflet + MapTiler |
| CI/CD | GitHub Actions |

## Local Development Setup

### Prerequisites

- Node.js 20+
- Python 3.12+
- Docker (for backend containerized development)
- GDAL 3.0+ (for local ortho processing without Docker)
- MongoDB instance (local or Atlas)
- Azure Blob Storage account

### Frontend (apps/web)

```bash
# From repository root
npm ci
npm run dev:web
```

The frontend dev server runs at `http://localhost:4321/cloud-viewer`.

### Backend (apps/api)

```bash
# Option 1: Docker (recommended)
docker-compose -f apps/api/docker-compose.yml up --build

# Option 2: Local Python
pip install -r apps/api/requirements.txt
cd apps/api && uvicorn main:app --reload --port 8000
```

The API runs at `http://localhost:8000` with interactive docs at `/docs`.

### Full Stack

```bash
npm run dev
```

Runs both frontend and backend concurrently.

## Environment Variables

### Backend (apps/api/.env)

| Variable | Required | Description |
|----------|----------|-------------|
| `MONGO_CONNECTION_STRING` | Yes | MongoDB/Cosmos DB connection string |
| `AZURE_STORAGE_CONNECTION_STRING` | Yes | Azure Blob Storage connection string |
| `NAME` | Yes | Database name and blob container name (e.g., `hwc-potree`) |
| `PORT` | No | HTTP port (default: `8000`) |
| `POTREE_PATH` | No | Path to PotreeConverter binary (default: `/app/bin/PotreeConverter`) |

### Frontend (apps/web/.env)

| Variable | Required | Description |
|----------|----------|-------------|
| `PUBLIC_API_BASE_URL` | Yes | Backend API URL (e.g., `https://hwc-potree-api.azurecontainerapps.io`) |
| `PUBLIC_MAPTILER_API_KEY` | Yes | MapTiler API key for map tiles |
| `PUBLIC_APP_BASE_PATH` | No | Base path for the app (default: `/cloud-viewer`) |

### GitHub Secrets (for CI/CD)

| Secret | Description |
|--------|-------------|
| `AZURE_CREDENTIALS` | Azure service principal JSON for CLI login |
| `AZURE_RESOURCE_GROUP` | Azure resource group name |
| `AZURE_STATIC_WEB_APPS_TOKEN` | Deployment token for Azure Static Web Apps |
| `MONGO_CONNECTION_STRING` | Production MongoDB connection string |
| `AZURE_STORAGE_CONNECTION_STRING` | Production Azure Blob Storage connection string |

### GitHub Variables (for CI/CD)

| Variable | Description |
|----------|-------------|
| `AZURE_MONGO_NAME` | Database/container name (e.g., `hwc-potree`) |
| `PUBLIC_MAPTILER_API_KEY` | MapTiler API key |
| `AZURE_CONTAINER_APP_NAME` | Container App name (e.g., `hwc-potree-api`) |
| `AZURE_CONTAINER_ENV_NAME` | Container App Environment name |
| `AZURE_STATIC_WEB_APP_NAME` | Static Web App resource name |

## Build and Run Commands

```bash
# Install all dependencies (root + workspaces)
npm ci

# Development
npm run dev              # Full stack (frontend + backend)
npm run dev:web          # Frontend only
npm run dev:api          # Backend only (requires Python env)

# Build
npm run build            # Build frontend for production

# Utilities
npm run assets:sync      # Sync branding assets to app public dirs
npm run potree:sync      # Sync Potree library to web public dir
```

## CI/CD Overview

### Branching Strategy

```
feature/* ──► dev ──► main
              │         │
              │         └── production.yml (deploy to Azure)
              └── dev.yml (lint, build, validate)
```

- `feature/*` branches: individual work, PR into `dev`
- `dev`: integration branch, CI validates on push/PR
- `main`: production branch, triggers full deployment

### dev.yml (CI only)

Triggers on push to `dev` or PRs targeting `dev`. Runs:
1. Install dependencies
2. Lint/type-check frontend
3. Build frontend
4. Validate backend (syntax check, import validation)

No deployment occurs.

### production.yml (CD)

Triggers on push to `main`. Sequential jobs:
1. Build backend Docker image and push to GHCR
2. Deploy/update Azure Container App with new image
3. Retrieve backend FQDN dynamically via Azure CLI
4. Build frontend with discovered `PUBLIC_API_BASE_URL`
5. Deploy frontend to Azure Static Web Apps

The frontend build happens AFTER backend deployment so the API URL is auto-discovered — no manual variable updates or redeployments needed.

## Infrastructure Setup (From Scratch)

### Required Azure Resources

1. **Resource Group** — logical container for all resources
2. **Azure Cosmos DB for MongoDB** (or MongoDB Atlas) — project/job metadata
3. **Azure Blob Storage Account** — point cloud files, ortho files, thumbnails (public blob access)
4. **Azure Container Apps Environment** — hosting environment for backend
5. **Azure Container App** — the backend API service
6. **Azure Static Web App** — frontend hosting

### Azure CLI Provisioning

```bash
# Create resource group
az group create --name hwc-project-rg --location eastus

# Create Cosmos DB (MongoDB API)
az cosmosdb create --name hwc-cosmos --resource-group hwc-project-rg --kind MongoDB
az cosmosdb mongodb database create --account-name hwc-cosmos --resource-group hwc-project-rg --name hwc-potree

# Create Storage Account
az storage account create --name hwcstorage --resource-group hwc-project-rg \
  --sku Standard_LRS --allow-blob-public-access true
az storage container create --name hwc-potree --account-name hwcstorage --public-access blob

# Create Container Apps Environment
az containerapp env create --name hwc-container-env --resource-group hwc-project-rg --location eastus

# Create Container App (initial)
az containerapp create \
  --name hwc-potree-api \
  --resource-group hwc-project-rg \
  --environment hwc-container-env \
  --image ghcr.io/mafalana/hwc-project-2/api:latest \
  --target-port 8000 \
  --ingress external \
  --min-replicas 0 --max-replicas 3 \
  --cpu 2.0 --memory 4.0Gi

# Create Static Web App
az staticwebapp create --name hwc-cloud-viewer --resource-group hwc-project-rg --location eastus2
```

### GHCR Setup

GitHub Container Registry uses the built-in `GITHUB_TOKEN` — no additional secrets needed for push/pull within the same repository. The production workflow authenticates automatically.

### GitHub Secrets Configuration

Create an Azure service principal:

```bash
az ad sp create-for-rbac --name "hwc-github-actions" \
  --role contributor \
  --scopes /subscriptions/{subscription-id}/resourceGroups/hwc-project-rg \
  --sdk-auth
```

Add the JSON output as `AZURE_CREDENTIALS` in repository Settings → Secrets → Actions.

## Deployment Details

### Environment Separation

| Environment | Branch | Backend | Frontend |
|-------------|--------|---------|----------|
| Production | `main` | Azure Container App (auto-scaled) | Azure Static Web App |
| Development | `dev` | Local Docker / shared dev instance | Local dev server |

### How API URL Auto-Discovery Works

The production workflow eliminates manual API URL management:

1. Backend job deploys the container and retrieves the FQDN:
   ```bash
   az containerapp show --name $APP --resource-group $RG \
     --query "properties.configuration.ingress.fqdn" -o tsv
   ```
2. The FQDN is passed as a job output to the frontend job.
3. Frontend job builds with `PUBLIC_API_BASE_URL=https://{fqdn}`.

No manual GitHub Variable updates or redeployments required.

### Rollback Guidance

**Backend:**
```bash
# List recent revisions
az containerapp revision list --name hwc-potree-api --resource-group hwc-project-rg -o table

# Route traffic to a previous revision
az containerapp ingress traffic set --name hwc-potree-api --resource-group hwc-project-rg \
  --revision-weight <previous-revision>=100
```

**Frontend:** Re-run the production workflow on a previous commit, or revert the commit on `main`.

**Database:** Cosmos DB supports point-in-time restore via Azure Portal.

## Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| Frontend shows "API unavailable" | `PUBLIC_API_BASE_URL` wrong or backend down | Check `/health` endpoint; verify URL in build logs |
| Backend health check fails | MongoDB or Blob Storage unreachable | Verify connection strings in Container App env vars |
| Docker build fails on GDAL | GDAL version mismatch | Pin GDAL version in Dockerfile to match system package |
| PotreeConverter "not found" | Binary not executable or wrong path | Ensure `chmod +x` in Dockerfile; check `POTREE_PATH` |
| Job stuck in "processing" | Worker crashed mid-job | Restart container (stale jobs auto-reset on startup) |
| CORS errors in browser | Backend CORS misconfigured | Currently allows all origins (`*`) — verify middleware |
| Container App not starting | Image pull failure from GHCR | Check package visibility; verify image tag exists |
| Static Web App 404 | Base path mismatch | Ensure `PUBLIC_APP_BASE_PATH=/cloud-viewer` is set |
| Ortho upload fails | Invalid GeoTIFF or GDAL missing | Verify with `gdalinfo`; check container has GDAL installed |

## Contributing

### Branching Workflow

1. Create a feature branch from `dev`: `feature/your-feature-name`
2. Make changes, commit with conventional commit messages
3. Push and open a PR targeting `dev`
4. CI must pass before merge
5. After testing on `dev`, merge `dev` → `main` for production deploy

### Commit Naming Convention

Use [Conventional Commits](https://www.conventionalcommits.org/):

```
feat: add ortho upload progress bar
fix: resolve thumbnail generation for single-band TIFFs
chore: update dependencies
docs: add API endpoint documentation
refactor: extract job polling into shared hook
```

### Pull Request Expectations

- CI passes (lint, build, type-check)
- No hardcoded secrets or personal URLs
- New API endpoints include basic validation
- Frontend changes tested in local dev server
- Breaking changes documented in PR description

### Formatting/Linting

- Frontend: TypeScript strict mode (Astro tsconfig)
- Backend: Python standard formatting (ruff or black recommended)
- LF line endings, no trailing whitespace

### Before Merging to Main

- Verify `npm run build` succeeds
- Test backend changes with `docker-compose up --build`
- Verify `/health` endpoint responds correctly
- Test affected frontend pages in browser

## Known Issues and Flags

See [HANDOFF_ISSUES.md](./HANDOFF_ISSUES.md) for a complete list of issues identified during handoff preparation, including:
- Stale documentation references
- Security concerns
- Missing configurations
- Hardcoded values requiring update
- Dependency concerns

## License

Proprietary — HWC Development Team
