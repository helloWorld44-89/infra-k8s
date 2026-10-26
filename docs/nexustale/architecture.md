# NexusTale Architecture

## Overview

NexusTale is a personal knowledge management / note-taking application with a React frontend and FastAPI backend. Uses PostgreSQL with pgvector for embeddings, Redis for caching/sessions.

## Components

| Component | Technology | Purpose |
|---|---|---|
| `nexustale-api` | Python/FastAPI (ghcr.io/helloworld44-89/nexustale/api:v0.3.2) | Backend API |
| `nexustale-frontend` | React/Vite (ghcr.io/helloworld44-89/nexustale/frontend:v0.3.2) | SPA frontend |
| `nexustale-redis` | Redis 7 (redis:7-alpine) | Sessions, caching, Celery broker |
| `nexustale-db` | PostgreSQL + pgvector (CNPG shared-postgres) | Primary DB + vector embeddings |

## Data Flow

```
Browser
    │
    ▼ HTTPS
Traefik Ingress
    │
    ├──▶ Frontend (static) → nexustale-frontend (port 80)
    │
    ▼ API calls
Backend (port 8000) → nexustale-api
    │
    ├──▶ PostgreSQL (shared-postgres cluster)
    ├──▶ Redis (sessions, cache, Celery)
    └──▶ pgvector (embeddings for search/RAG)
```

## Network

- **Frontend Ingress**: `nexustale.conderhome.xyz` → Traefik → Frontend (port 80)
- **API Ingress**: `nexustale.conderhome.xyz/api/*` → Traefik → API (port 8000)
- **TLS**: Wildcard cert `wildcard-conderhome-xyz-tls`
- **Internal**:
  - Frontend → API: HTTP on port 8000
  - API → PostgreSQL: 5432 (via CNPG service)
  - API → Redis: 6379

## Storage

| Component | Storage | Details |
|---|---|---|
| PostgreSQL | CNPG `shared-postgres` | Shared cluster, dedicated DB `nexustale` |
| Redis | PVC `nexustale-redis` (1Gi, local-path) | Sessions, cache, Celery |
| Frontend | None (static) | Served by nginx in container |

## Database

- **CNPG Cluster**: `shared-postgres` (in `databases` namespace)
- **Database**: `nexustale` (dedicated, not shared with other apps)
- **Extensions**: `pgvector` (pre-installed in v0.3.2 image)
- **Migrations**: Alembic (auto-run on startup, current: v40)
- **User**: `nexustale` (non-superuser, owns DB)

## Redis

- **Image**: `redis:7-alpine`
- **PVC**: `nexustale-redis` (1Gi, local-path)
- **Usage**: Sessions, rate-limiting cache, Celery broker/result backend

## Security

- **Auth**: JWT tokens (access + refresh)
- **Passwords**: bcrypt (handled by FastAPI-Users)
- **CORS**: Configured for `nexustale.conderhome.xyz`
- **Rate Limiting**: SlowAPI on auth endpoints
- **Secrets**: `nexustale-secrets` (JWT secret, DB password, Redis password)

## Resources

| Component | CPU Request | CPU Limit | Memory Request | Memory Limit |
|---|---|---|---|---|
| api | 100m | 1000m | 256Mi | 1Gi |
| frontend | 20m | 200m | 64Mi | 256Mi |
| redis | 50m | 500m | 128Mi | 512Mi |

## Health Checks

| Component | Endpoint |
|---|---|
| API | `GET /health` |
| Frontend | `GET /` (nginx) |
| Redis | `PING` |

## Version

- **Current**: v0.3.2 (fixed MinIO removal, pgvector pre-installed)
- **Migrations**: v40 (clean from scratch on CNPG)
- **Previous**: v0.3.1 had MinIO dependency, broken migration 37