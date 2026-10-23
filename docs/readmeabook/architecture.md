---
title: ReadMeABook Architecture Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [architecture, documentation, audiobook, kubernetes]
---

# ReadMeABook Architecture Guide

## Overview

ReadMeABook is a self-hosted audiobook request and acquisition engine — "Radarr/Sonarr + Overseerr, but for audiobooks." Users browse or swipe through AI-curated recommendations (BookDate), request a title, and ReadMeABook handles the rest: searching indexers via Prowlarr, downloading via qBittorrent/SABnzbd, merging multi-file downloads into chapterized M4B files, and importing the result into Plex or Audiobookshelf. In this cluster it runs as a single all-in-one container (app + embedded Postgres + Redis) in the `media` namespace, serving the household at `https://audiobook.conderhome.xyz`.

## System Architecture

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│               readmeabook Pod (media namespace)                  │
│                                                                    │
│  ┌───────────────┐   ┌───────────────┐   ┌─────────────────────┐ │
│  │  Web UI /      │   │  Request /    │   │  BookDate           │ │
│  │  Admin Panel   │   │  Approval     │   │  Recommendation     │ │
│  │  (port 3030)   │   │  Engine       │   │  Engine (AI)        │ │
│  └───────────────┘   └───────────────┘   └─────────────────────┘ │
│                                                                    │
│  ┌───────────────┐   ┌───────────────┐   ┌─────────────────────┐ │
│  │  PostgreSQL    │   │  Redis        │   │  Chapter Merge /     │ │
│  │  (local data)  │   │  (cache/jobs) │   │  Conversion Engine   │ │
│  └───────────────┘   └───────────────┘   └─────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
               │                     │                    │
               ▼                     ▼                    ▼
        ┌────────────┐        ┌────────────┐      ┌───────────────┐
        │  Prowlarr   │        │ qBittorrent│      │  SABnzbd       │
        │ (indexers)  │        │ (torrents) │      │  (usenet)      │
        └────────────┘        └────────────┘      └───────────────┘
               │                                            │
               ▼                                            ▼
        ┌───────────────────────── downloads-pvc (NFS) ───────────┐
        │              finished downloads land here                │
        └───────────────────────────────────────────────────────────┘
                                   │
                                   ▼
        ┌───────────────────────── audiobooks-pvc (NFS) ───────────┐
        │   organized, chapterized library consumed by Plex /       │
        │              Audiobookshelf                               │
        └───────────────────────────────────────────────────────────┘
```

Unlike AudioFile (a separate SQLite-based audiobook *converter/manager*), ReadMeABook is purely an *acquisition* layer — it does not stream audio itself; playback stays with Plex or Audiobookshelf once files land on the shared media PVC.

### Core Components

#### 1. Application Container

- Single unified image: `ghcr.io/kikootwo/readmeabook:latest`
- Bundles the web app, API, PostgreSQL, and Redis in one container (`dockerfile.unified` upstream)
- Listens on port **3030** inside the pod
- Deployment strategy is `Recreate` (not `RollingUpdate`) — appropriate since the embedded Postgres/Redis data directories are single-writer, ReadWriteOnce volumes that cannot be attached to two pods at once

#### 2. Data Layer (self-contained, not the shared CNPG cluster)

Unlike most other apps in this cluster, ReadMeABook does **not** use the shared CloudNativePG Postgres cluster. It ships and manages its own Postgres and Redis inside the pod, each backed by a dedicated `local-path` PVC:

- **PostgreSQL** (`readmeabook-pgdata`, 5Gi, local-path) — mounted at `/var/lib/postgresql/data`; stores requests, users, settings, BookDate swipe history
- **Redis** (`readmeabook-redis`, 1Gi, local-path) — mounted at `/var/lib/redis`; job/cache state

This is a deliberate isolation choice: the app bundles its own DB versions and schema migrations, so coupling it to the shared CNPG cluster would add migration risk for no real benefit, at the cost of it living outside the cluster's centralized backup/HA story for Postgres.

#### 3. Request & Approval Engine

- Users search for audiobooks (Audible-backed metadata) and submit requests
- Multi-user admin approval workflow gates requests before download (configurable per deployment; single-user homelab use typically runs with approval disabled or self-approved)
- **BookDate**: Tinder-style swipe UI over AI-generated recommendations (OpenAI/Claude/local LLM backends) — swiping right creates a request

#### 4. Acquisition Pipeline

- **Prowlarr**: indexer aggregation/search for both torrents and NZBs (external service, not bundled — ReadMeABook calls out to it)
- **qBittorrent**: torrent downloads
- **SABnzbd**: usenet/NZB downloads
- Downloaded files land on `downloads-pvc` (NFS-backed, shared with other media apps)

#### 5. Post-Processing

- **Chapter merging**: multi-file downloads (e.g. one file per chapter) are merged into a single M4B with embedded chapter markers
- **E-book sidecar**: optional EPUB/PDF companion download from a shadow library source
- Organized output is written to `audiobooks-pvc` (NFS-backed `/media` mount) for Plex/Audiobookshelf to pick up

#### 6. Config, Cache & Setup

- `config` volume (`readmeabook-config`, 1Gi, local-path, `/app/config`): app settings, OIDC config, API keys for Plex/Prowlarr/qBittorrent/SABnzbd
- `cache` volume (`readmeabook-cache`, 5Gi, local-path, `/app/cache`): working directory for downloads-in-progress, chapter merge scratch space, cover art cache
- **Setup wizard**: guided first-run configuration with live connection testing against Plex/Audiobookshelf, Prowlarr, and the download clients

### Data Flow Architecture

#### Request → Library Pipeline

```
1. User search/swipe → 2. Create request → 3. (Admin approval, if enabled)
       ↓
4. Prowlarr indexer search → 5. Download via qBittorrent/SABnzbd → 6. Land on downloads-pvc
       ↓
7. Chapter merge / M4B conversion → 8. Optional e-book sidecar fetch
       ↓
9. File organization on audiobooks-pvc → 10. Plex/Audiobookshelf library scan/import
```

### Networking & Ingress

- **Service**: ClusterIP `readmeabook`, port `http` → container port `3030`
- **Ingress**: Traefik (`className: traefik`), host `audiobook.conderhome.xyz`, path `/` (Prefix)
- **TLS**: shared wildcard certificate `wildcard-conderhome-xyz-tls` (not a per-app cert-manager Certificate — reuses the cluster's wildcard `*.conderhome.xyz` cert)
- No external OAuth/SSO proxy in front of it at the ingress level; app-level OIDC is supported but configured in-app (not inspected here)

### Storage Architecture

| Volume | PVC | Size | StorageClass | Mount | Purpose |
|--------|-----|------|---------------|-------|---------|
| config | `readmeabook-config` | 1Gi | local-path | `/app/config` | App settings, integration credentials |
| cache | `readmeabook-cache` | 5Gi | local-path | `/app/cache` | Download/processing scratch space |
| pgdata | `readmeabook-pgdata` | 5Gi | local-path | `/var/lib/postgresql/data` | Dedicated Postgres data directory |
| redis | `readmeabook-redis` | 1Gi | local-path | `/var/lib/redis` | Dedicated Redis data directory |
| downloads | `downloads-pvc` (existing claim, NFS, 500Gi RWX) | — | — | `/downloads` | Shared raw download landing zone |
| media | `audiobooks-pvc` (existing claim, NFS, 500Gi RWX) | — | — | `/media` | Shared organized audiobook library |

`local-path` volumes are node-local, so the pod is implicitly pinned to whatever node currently holds those PVs (observed running on `prod-k3s-3`); it cannot be rescheduled to another node without the data moving too. The `downloads` and `media` claims are NFS-backed and RWX, so they're portable across nodes and shared with sibling media apps (e.g. a *arr stack, Plex).

### Health & Probes

- **Liveness**: `GET /api/health` on port 3030, 60s initial delay, every 30s, 3 failures to restart
- **Readiness**: `GET /api/health` on port 3030, 5s initial delay, every 10s

### Resource Profile

| | Requests | Limits |
|---|----------|--------|
| CPU | 100m | 1000m (1 core) |
| Memory | 512Mi | 2Gi |

Observed live usage (via `kubectl top`): ~3m CPU, ~374Mi memory at idle — well under requests, consistent with a lightly-used single-household deployment where most load is bursty (download/transcode activity) rather than constant.

## Technology Stack

- **Container**: single unified image (`ghcr.io/kikootwo/readmeabook`), app + embedded Postgres + embedded Redis
- **Database**: PostgreSQL (self-managed, not the shared CNPG cluster)
- **Cache/jobs**: Redis (self-managed)
- **Download clients**: qBittorrent (torrents), SABnzbd (usenet) — external, not bundled
- **Indexer aggregation**: Prowlarr — external, not bundled
- **Media servers**: Plex and/or Audiobookshelf — external, consume the finished library
- **AI recommendations**: OpenAI / Claude / local LLM backends for BookDate
- **Orchestration**: Flux HelmRelease using the `bjw-s` `app-template` chart (v3.x) on k3s
- **Ingress**: Traefik, shared wildcard TLS cert
- **License**: AGPL-3.0 (upstream project)

## Deployment Isolation Notes

- This is one of the few apps in the cluster running its *own* Postgres and Redis instead of the shared CNPG Postgres cluster or a shared Redis — a conscious trade-off for app simplicity over central backup/HA coverage. Backups of `readmeabook-pgdata` must be handled explicitly (see Deployment Guide) since it is outside the cluster's CNPG backup routine.
- All persistent state outside the two shared NFS media PVCs lives on `local-path` (node-local) storage, so losing the node `prod-k3s-3` disk means losing config/cache/pgdata/redis unless backed up separately.
