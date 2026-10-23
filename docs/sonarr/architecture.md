---
title: Sonarr Architecture Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [architecture, documentation, tv, media-automation, kubernetes]
---

# Sonarr Architecture Guide

## Overview

Sonarr is a self-hosted TV show collection manager and PVR (Personal Video Recorder) for Usenet and BitTorrent users. It monitors multiple RSS feeds and indexers for new episodes of tracked series, automatically downloads, sorts, and renames them, and can upgrade existing files as better quality releases become available. In this homelab it runs as part of a wider media automation stack alongside **Radarr** (movies), **Prowlarr** (indexer manager), and **FlareSolverr** (Cloudflare/CAPTCHA bypass proxy for indexers), all deployed in the `media` namespace of the k3s cluster.

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Sonarr Pod (media ns)                  │
│                                                               │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────┐  │
│  │   .NET / ASP.NET │  │   React Web UI  │  │  SQLite    │  │
│  │   Core Backend   │  │   (served by    │  │  Database  │  │
│  │                  │  │    backend)     │  │            │  │
│  └─────────────────┘  └─────────────────┘  └────────────┘  │
│                                                               │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────┐  │
│  │  RSS/Indexer     │  │  Download Client│  │  Naming/   │  │
│  │  Sync Scheduler  │  │  Connectors     │  │  Import    │  │
│  │                  │  │  (SABnzbd/      │  │  Engine    │  │
│  │                  │  │   qBittorrent)  │  │            │  │
│  └─────────────────┘  └─────────────────┘  └────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Core Components

#### 1. Backend Service

**.NET / ASP.NET Core Backend**
- Monolithic server process (the `linuxserver/sonarr` image bundles .NET runtime + app)
- Serves both the REST API (`/api/v3`) and the bundled web UI on a single port
- Owns all business logic: series tracking, episode calendar, quality decisions, renaming
- Single-process design — no separate worker/queue service like Celery/Redis in AudioFile

**Database Layer**
- **SQLite**: `/config/sonarr.db` stores series, episodes, history, download queue, indexers, and settings
- File-based, backed up by snapshotting the `config` PersistentVolumeClaim
- Internal ORM/ADO.NET data layer (no external DB dependency)

**Scheduler**
- Internal task scheduler (no Celery/Redis equivalent) drives:
  - RSS sync (indexer polling, interval configurable per indexer)
  - Series/episode metadata refresh
  - Health checks and disk space checks
  - Automatic search for missing/cutoff-unmet episodes

#### 2. Frontend Application

**Bundled Web UI**
- React-based SPA served directly by the Sonarr backend process (no separate frontend container/pod)
- Series library grid/list views, calendar, activity queue, and settings panels
- Communicates with the backend exclusively through the same `/api/v3` REST API available externally

#### 3. Indexer & Download Integration Layer

**Indexer Clients**
- Sonarr queries configured indexers (Usenet/Torrent trackers) for episode releases
- In this deployment, indexer configuration is centrally managed by **Prowlarr**, which syncs indexer definitions and API keys into Sonarr via Prowlarr's "Sonarr application" integration — Sonarr itself does not need indexers added manually
- **FlareSolverr** is used as an indexer proxy for trackers that sit behind Cloudflare's JS challenge/CAPTCHA; Prowlarr routes the relevant indexer's requests through FlareSolverr, and Sonarr benefits transparently since it only talks to indexers through Prowlarr

**Download Client Connectors**
- Sonarr pushes accepted releases to a configured download client (SABnzbd, NZBGet, qBittorrent, Deluge, etc. — whichever is deployed elsewhere in the cluster) via each client's native API
- Polls the download client for completion status and queue state

#### 4. File Management / Import Engine

**Media Naming & Import**
- Watches download client "complete" folders (mounted from the shared `downloads-pvc`) for finished episode files
- Matches downloaded files to tracked series/episodes using release parsing (SxxEyy patterns, scene naming conventions)
- Applies configurable naming templates and hardlinks/moves files into the TV library path
- Performs quality upgrade logic: replaces existing episode files when a better-quality release is imported (per quality profile/cutoff)

**Metadata Service**
- **TheTVDB**: Primary metadata source for series, season, and episode information
- **SkyHook** (Sonarr's own metadata proxy service): Intermediary that aggregates TheTVDB (and historically TVMaze) data for the application
- Automatic series/episode metadata refresh on a scheduled interval

### Data Flow Architecture

#### Episode Acquisition Pipeline

```
1. RSS/Indexer Sync → 2. Release Grab Decision → 3. Send to Download Client
     ↓                        ↓                           ↓
4. Download Client Polling → 5. Import/Rename on Complete → 6. Library Update → 7. Notification
```

#### Import Process

1. **Detect Completed Download**: Download client reports completion, or Sonarr scans the downloads path
2. **Parse Release**: Extract series, season, episode, quality, and release group from filename
3. **Match to Library**: Associate parsed release with a tracked series/episode in the database
4. **Quality Decision**: Compare against existing file quality/cutoff; decide import vs. reject/upgrade
5. **File Operation**: Hardlink or move file into the TV library using the configured naming format
6. **Database Update**: Record episode file, history entry, and update series statistics
7. **Notifications**: Fire configured notification connections (Discord, webhook, etc.) if enabled

### API Architecture

#### REST API Endpoints (v3)

Sonarr exposes a versioned REST API under `/api/v3`. Representative resource groups:

**Series Management**
- `GET /api/v3/series` — List all tracked series
- `GET /api/v3/series/{id}` — Get series details
- `POST /api/v3/series` — Add a new series to monitor
- `PUT /api/v3/series/{id}` — Update series (monitored state, quality profile, paths)
- `DELETE /api/v3/series/{id}` — Remove a series

**Episode Management**
- `GET /api/v3/episode` — List episodes (filterable by series/season)
- `GET /api/v3/episode/{id}` — Episode detail
- `PUT /api/v3/episode/monitor` — Toggle monitored status in bulk

**Queue & Activity**
- `GET /api/v3/queue` — Current download queue
- `GET /api/v3/history` — Grab/import history
- `POST /api/v3/command` — Trigger commands (RSS sync, series search, rename, refresh)

**Indexer & Download Client Config**
- `GET /api/v3/indexer` — List configured indexers (managed by Prowlarr in this deployment)
- `GET /api/v3/downloadclient` — List configured download clients

See `api-documentation.md` for full request/response examples.

### Security Architecture

#### Authentication & Authorization
- **Form/Basic Auth**: Configurable in Settings → General (username/password), disabled by default on trusted LANs
- **API Key**: A single, per-instance API key (Settings → General → Security) required for all REST API calls; also usable by Prowlarr/other *arr apps for application integration
- **Reverse Proxy**: Exposed externally only via the Traefik ingress with TLS termination (`sonarr.conderhome.xyz`) — no direct NodePort exposure

#### Data Security
- Config, API key, and SQLite database persist on the `sonarr-config` PVC (`/config`)
- No secrets are templated into the HelmRelease directly; the API key is generated and stored inside `/config/config.xml` on first boot
- Container runs as the `linuxserver` image's standard PUID/PGID model (default root in this chart; no explicit `securityContext` overrides are set)

### Deployment Architecture

#### Containerization
- **Image**: `linuxserver/sonarr` (LinuxServer.io maintained image), currently pinned to `4.0.20`
- **Helm Chart**: `bjw-s` `app-template` (v3.x) — a generic templated chart used across this cluster's `media` namespace apps
- **GitOps**: Managed by Flux (`HelmRelease` + image automation `$imagepolicy` marker for the `flux-system:sonarr:tag` policy, enabling automated tag bumps)

#### Orchestration (k3s / Flux)
- Single `Deployment` with `strategy: Recreate` (SQLite-backed, single-writer apps can't do rolling updates safely)
- `Service` exposing port `8989` (named `http`)
- `Ingress` (Traefik) routing `sonarr.conderhome.xyz` → service, TLS via the cluster's wildcard certificate (`wildcard-conderhome-xyz-tls`)
- `PersistentVolumeClaim` for `/config` (1Gi, ReadWriteOnce, dedicated `sonarr-config` claim)
- Shared PVCs mounted from other apps: `downloads-pvc` → `/downloads`, `tv-pvc` → `/tv/TV Shows`

#### Resource Management
- No explicit CPU/memory `resources` block is currently set on the Sonarr container (unlike Radarr/Prowlarr siblings in this stack, which do define requests/limits) — see deployment-guide.md for recommended values
- Storage: large TV library and downloads volumes are shared PVCs, decoupled from the small config volume

### Integration Points

#### Media Automation Stack
- **Prowlarr** (`prowlarr.conderhome.xyz`, namespace `media`): Central indexer manager. Prowlarr pushes indexer configuration and API sync into Sonarr (and Radarr) via each app's "Applications" integration, so indexers are added/maintained in one place instead of per-app.
- **FlareSolverr** (`media` namespace, internal service only, port `8191`): A headless-browser-based proxy that solves Cloudflare JS challenges/CAPTCHAs on behalf of indexers that require it. Prowlarr is configured to route specific indexers through FlareSolverr; Sonarr never talks to FlareSolverr directly — the chain is Sonarr → Prowlarr-synced indexer → (optionally) FlareSolverr.
- **Radarr** (`radarr.conderhome.xyz`, namespace `media`): The movie equivalent of Sonarr, same `app-template` chart pattern, same `downloads-pvc` for completed downloads, separate `movies-pvc` for the movie library. Sonarr and Radarr are independent instances with no direct API relationship to each other, but they share the download client and downloads volume.
- **Download Client** (external to this chart): Configured inside Sonarr's Settings → Download Clients; shares the `downloads-pvc` volume so Sonarr can see completed files without network file transfer.

#### File System Integration
- `/tv/TV Shows` — root library path where imported/renamed episodes live (backed by `tv-pvc`)
- `/downloads` — shared completed-download staging area (backed by `downloads-pvc`, shared with Radarr and the download client)
- `/config` — Sonarr's own database, logs, and settings (backed by dedicated `sonarr-config` PVC)

### Monitoring & Observability

#### Health Monitoring
- Sonarr exposes `/ping` (unauthenticated) and `/api/v3/health` (authenticated) for health/status checks
- No liveness/readiness probes are currently defined in the HelmRelease — recommended addition, see deployment-guide.md
- Flux reports `HelmRelease` reconciliation status via `flux get helmreleases -n media`

#### Logging
- Application logs written to `/config/logs/` inside the pod (persisted on the config PVC)
- Retrievable via `kubectl logs -n media deploy/sonarr` for live/recent output
- Log level configurable in Settings → General → Logging (Info/Debug/Trace)

### Performance Considerations

#### Concurrency
- Single-process .NET application — all RSS sync, import, and search operations are scheduled internally, not distributed across workers
- SQLite is single-writer; heavy concurrent import activity (e.g., large library scans) can serialize on the database

#### Scalability
- Not designed for horizontal scaling — `strategy: Recreate` and SQLite enforce a single replica
- Vertical scaling (CPU/memory limits) is the primary lever for performance tuning under heavy libraries

### Backup & Recovery

#### Database Backup
- Sonarr has a built-in backup feature (Settings → General → Backups) that snapshots the database and config to `/config/Backups`
- Cluster-level backup should additionally snapshot/back up the `sonarr-config` PVC directly

#### Media Backup
- TV library (`tv-pvc`) backup is handled at the storage layer, independent of Sonarr itself — Sonarr can always re-import/re-match files from a restored library path

## Technology Stack

### Backend Technologies
- **.NET (ASP.NET Core)**: Primary application runtime (bundled in the `linuxserver/sonarr` image)
- **SQLite**: Embedded database for all application state
- **SkyHook / TheTVDB**: Metadata provider services

### Frontend Technologies
- **React**: Bundled web UI, served by the same backend process (no separate frontend deployment)

### Infrastructure Technologies
- **Docker/containerd**: Container runtime (k3s)
- **Flux CD**: GitOps delivery of the `HelmRelease`
- **bjw-s app-template Helm chart**: Generic chart used for this and sibling apps in `media`
- **Traefik**: Ingress controller and TLS termination
- **k3s PersistentVolumeClaims**: Config and media storage

## Development Guidelines

### Code Structure
Sonarr itself is third-party (upstream project, not maintained in this repo). This repo only owns:
- `apps/media/sonarr/helmrelease.yml` — Helm values (image tag, service, ingress, persistence)
- `apps/media/sonarr/kustomization.yaml` — Kustomize wrapper referencing the HelmRelease

### Change Management
- Image tag bumps are automated via Flux image automation (`$imagepolicy` annotation) when enabled cluster-wide, or applied manually by editing the `tag:` field
- Any structural change (new ingress host, new persistence mount) is a plain YAML edit + `git commit` + Flux reconciliation — no manual `kubectl apply` required

### Documentation Standards
- API Documentation: see `api-documentation.md` for `/api/v3` endpoint reference
- User Guide: see `user-guide.md` for day-to-day usage
- Deployment Guide: see `deployment-guide.md` for install/upgrade/ops procedures

This architecture reflects a lightweight, single-process PVR application integrated into a shared media-automation stack, where cross-app coordination (indexers, Cloudflare bypass) is centralized through Prowlarr rather than configured redundantly in each `*arr` app.
