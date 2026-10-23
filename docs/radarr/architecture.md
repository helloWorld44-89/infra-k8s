---
title: Radarr Architecture Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [architecture, documentation, movies, kubernetes]
---

# Radarr Architecture Guide

## Overview

Radarr is a movie collection manager and automation tool that monitors indexers for desired movies, sends matching releases to a download client, and renames/organizes the resulting files into a managed movie library. In this cluster it runs as a single-replica Deployment in the `media` namespace, managed by Flux via a `HelmRelease` using the `bjw-s` `app-template` chart, and it is one part of a larger PVR (personal video recorder) automation stack alongside Sonarr (TV), Prowlarr (indexer manager), and FlareSolverr (Cloudflare/CAPTCHA bypass proxy).

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     media namespace (k3s)                       │
│                                                                   │
│  ┌──────────────┐   indexer sync    ┌──────────────┐            │
│  │   Prowlarr    │ ────────────────▶│   Radarr      │            │
│  │  (indexers)   │ ◀──── app sync ───│ (movies)      │            │
│  └──────┬───────┘                   └──────┬────────┘            │
│         │ proxies challenge pages            │ pushes releases     │
│         ▼                                     ▼                    │
│  ┌──────────────┐                    ┌──────────────┐            │
│  │ FlareSolverr  │                    │ Download      │            │
│  │ (CF bypass)   │                    │ Client (ext.) │            │
│  └──────────────┘                    └──────┬────────┘            │
│                                               │ completed files      │
│                                               ▼                    │
│                                       ┌──────────────┐            │
│                                       │ downloads-pvc │            │
│                                       │ movies-pvc    │            │
│                                       └──────────────┘            │
└─────────────────────────────────────────────────────────────────┘
```

### Core Components

#### 1. Radarr Application Container

**linuxserver/radarr**
- .NET-based monolithic application exposing a web UI and REST API v3 on port `7878`
- Single-process container (no sidecars) — LinuxServer.io image with `s6-overlay` init, running as the container's default user/group
- Maintains its own SQLite database for movie metadata, quality profiles, history, and queue state
- `TZ` environment variable (`America/Chicago`) controls scheduling and log timestamps

**Configuration Store**
- SQLite database and `config.xml` persisted under `/config`
- `config.xml` holds the instance API key, authentication mode, log level, and port/SSL settings
- No external database dependency — unlike AudioFile's Celery/Redis stack, Radarr is self-contained

#### 2. Indexer & Search Integration

**Prowlarr (indexer manager)**
- Radarr does not talk to torrent/usenet indexers directly; Prowlarr centralizes indexer configuration and syncs its indexer list into Radarr (and Sonarr) as "Prowlarr-managed" indexers via Radarr's API
- Radarr queries these synced indexers when searching for releases (manual search, RSS sync, or automatic search on new movie adds)

**FlareSolverr (Cloudflare/CAPTCHA bypass)**
- Several indexers sit behind Cloudflare's JS challenge or CAPTCHA pages
- Prowlarr is configured to route those specific indexers' requests through FlareSolverr (`http://flaresolverr.media.svc.cluster.local:8191`), which runs a headless browser to solve the challenge and return clean HTML/JSON
- Radarr itself never talks to FlareSolverr directly — the dependency is transitive through Prowlarr

**Sonarr (sibling PVR for TV)**
- Runs the same `app-template` pattern in the same namespace; shares the `downloads-pvc` volume with Radarr
- No direct API coupling to Radarr — they are independent consumers of Prowlarr's indexer sync and the shared download client/folder

#### 3. Download Client Integration

- Radarr itself does not download files; it pushes release selections (via their indexer-provided NZB/torrent links) to an external download client (e.g. qBittorrent/SABnzbd, configured in Radarr Settings → Download Clients, not part of this manifest)
- The download client writes completed files into the shared `downloads-pvc`, mounted at `/downloads` in the Radarr container
- Radarr polls/receives completion events and imports finished movies from `/downloads` into the organized library at `/movies`

#### 4. File Organization Engine

- **Import pipeline**: detects completed downloads → matches against the monitored movie → renames per the configured naming format → moves/hardlinks into `/movies`
- **Media management**: optional recycling bin, permissions handling, and "Unmonitored" vs "Monitored" movie states that gate automatic searching

### Data Flow Architecture

#### Movie Acquisition Pipeline

```
1. Add Movie (UI/API) → 2. Indexer Search (via Prowlarr-synced indexers)
     ↓                                    ↓
3. (Cloudflare indexer? → FlareSolverr solves challenge via Prowlarr)
     ↓
4. Release Selected → 5. Sent to Download Client → 6. Download Completes
     ↓
7. Radarr Detects Completion → 8. Import & Rename → 9. Move to /movies
     ↓
10. Library Updated, Notifications Fired (Discord/webhook/etc. if configured)
```

#### RSS / Automatic Search Loop

1. **RSS Sync**: periodic task polls synced indexers for new releases matching monitored movies
2. **Quality/Custom Format Scoring**: candidate releases are scored and the best eligible one is grabbed
3. **Queue**: grabbed release enters Radarr's download queue, tracked until import or failure
4. **Failed Handling**: failed/stalled downloads can be blocklisted and automatically re-searched

### API Architecture

#### REST API v3 (selected areas)

**Movie Management**
- `GET /api/v3/movie` — list all monitored/library movies
- `GET /api/v3/movie/{id}` — movie detail
- `POST /api/v3/movie` — add a movie to the library
- `PUT /api/v3/movie/{id}` — update monitoring/quality profile
- `DELETE /api/v3/movie/{id}` — remove from library

**Search & Queue**
- `POST /api/v3/command` — trigger commands (`MoviesSearch`, `RescanMovie`, `RefreshMovie`, etc.)
- `GET /api/v3/queue` — current download queue
- `GET /api/v3/history` — grab/import/failure history

**System**
- `GET /api/v3/system/status` — version, build, and runtime info
- `GET /api/v3/health` — internal health checks (indexer, download client, root folder issues)

See `api-documentation.md` for the full endpoint reference and authentication details.

### Security Architecture

#### Authentication & Authorization
- **Forms Authentication**: username/password login enforced for the web UI (`AuthenticationMethod: forms`, `AuthenticationRequired: Enabled` in `config.xml`)
- **API Key**: a per-instance key (stored in `config.xml`, Settings → General in the UI) required on all REST API requests
- **No RBAC**: Radarr has a single admin-level account model; it does not support multi-user roles like AudioFile

#### Network Exposure
- Internal ClusterIP Service on port `7878`, fronted by a Traefik Ingress at `radarr.conderhome.xyz` with TLS via the shared `wildcard-conderhome-xyz-tls` certificate
- No authentication is offloaded to an external OIDC/SSO proxy in the current manifest — Radarr's built-in forms auth is the access control boundary at the ingress

#### Data Security
- Config volume (`radarr-config`, 1Gi) holds the API key and credentials-adjacent settings; access to this PVC is equivalent to access to the Radarr admin account
- No secrets are templated into the HelmRelease — the API key is generated and stored by Radarr itself on first boot

### Deployment Architecture

#### Containerization
- **Image**: `linuxserver/radarr`, pinned tag `6.4.4`, tracked by Flux's image automation (`$imagepolicy: flux-system:radarr:tag` marker)
- **Chart**: `bjw-s/app-template` v3.x — a generic Helm chart used across this cluster's `media` apps (Sonarr, Prowlarr, FlareSolverr all use the same pattern)
- **Single container**: no init containers or sidecars; `linuxserver` base image handles PUID/PGID-style permission handling internally

#### Orchestration
- **Flux HelmRelease**: reconciled hourly (`spec.interval: 1h`); any manifest change in Git is applied automatically
- **Deployment strategy**: `Recreate` (set by the chart default for single-replica stateful-ish apps) — avoids two Radarr processes writing to the same SQLite database concurrently
- **Volumes**:
  - `config` → PVC `radarr-config` (1Gi, RWO) mounted at `/config`
  - `media` → existing PVC `downloads-pvc` mounted at `/downloads` (shared with Sonarr and the download client)
  - `movies` → existing PVC `movies-pvc` mounted at `/movies` (organized library, shared with Immich? No — dedicated to Radarr's movie library)

#### Resource Management
- No `resources.requests`/`limits` are currently set in the Radarr HelmRelease (unlike Prowlarr and FlareSolverr, which define explicit CPU/memory requests and limits) — Radarr runs under the node's default BestEffort QoS class
- Single replica; no HPA or autoscaling — Radarr is not designed for horizontal scaling (SQLite backend, stateful queue)

### Integration Points

#### Internal Services (media namespace)
- **Prowlarr**: indexer sync target/source — Radarr is added as an "Application" in Prowlarr, which pushes indexer config into Radarr via Radarr's API key
- **FlareSolverr**: indirect dependency via Prowlarr for Cloudflare-protected indexers
- **Sonarr**: sibling app, shares `downloads-pvc`, no direct API relationship
- **Download client**: external to this manifest (configured in-app), writes to the shared downloads volume

#### External Services
- **Indexers**: torrent/usenet indexers configured in Prowlarr and synced to Radarr
- **Metadata providers**: Radarr's own metadata server (TMDb-backed) for movie search, posters, and information lookups
- **Notification targets**: optional Discord/webhook/Slack notifications configured in Radarr Settings → Connect (not present in current manifest; configured in-app if used)

### Monitoring & Observability

#### Health Monitoring
- `GET /api/v3/health` — in-app health checks surfaced in the UI (indexer errors, download client unreachable, root folder missing, etc.)
- No Kubernetes `livenessProbe`/`readinessProbe` is currently defined on the Radarr container — pod health is inferred purely from process liveness
- `kubectl get pods -n media` / `kubectl describe pod` are the operational tools for checking container state, since there is no HTTP probe wired into the Deployment

#### Logging
- Application logs written to `/config/logs` inside the container, retrievable via `kubectl logs` or by browsing the config PVC
- `LogLevel` is set to `debug` in the current `config.xml`, which is verbose — useful for troubleshooting indexer/import issues but generates larger log volume

### Performance Considerations

- **Single-threaded-ish import**: Radarr processes imports and searches largely serially per task type; concurrency is bounded by its own internal task scheduler, not by Kubernetes resources
- **SQLite contention**: the `Recreate` deployment strategy exists specifically to avoid two processes contending for the same database file
- **Shared storage I/O**: `downloads-pvc` is shared with Sonarr and the download client, so disk I/O contention there affects import speed for both PVRs

### Backup & Recovery

- **Config backup**: the `radarr-config` PVC contains the entire application state (database, API key, settings) — back this volume up to recover a working Radarr instance without re-adding movies or re-configuring indexers
- **Movie library**: the `movies-pvc` holds the actual media files; back up separately from config since it is typically much larger
- Radarr has an in-app backup feature (Settings → General → Backups) that snapshots the database and config to `/config/Backups` on a schedule

### Configuration Management

- **In-app settings** (not templated via Helm values): download clients, indexers (via Prowlarr sync), quality profiles, naming formats, notification connections, root folders
- **Helm-managed settings**: image tag, `TZ`, ingress host/TLS, PVC sizes/claims, service port
- Changing persistent settings (API key, auth mode) requires editing `config.xml` directly on the PVC or via the Radarr UI — these are not exposed as Helm values in this chart's current configuration

## Technology Stack

### Application
- **.NET (C#)**: Radarr's core runtime (via the `linuxserver/radarr` image, itself built on `linuxserver/baseimage-*` with `s6-overlay`)
- **SQLite**: embedded database for library, queue, and history state
- **React**: Radarr's web UI frontend (bundled in the image)

### Infrastructure Technologies
- **Kubernetes (k3s)**: orchestration platform
- **Flux CD**: GitOps reconciliation (`HelmRelease` + image automation)
- **Helm (`bjw-s/app-template`)**: generic chart used for Radarr, Sonarr, Prowlarr, and FlareSolverr
- **Traefik**: ingress controller terminating TLS and routing `radarr.conderhome.xyz`
- **Longhorn/local-path (cluster default)**: backing storage class for the `radarr-config` PVC

## Development Guidelines

### Code Structure
- Radarr is third-party upstream software (not developed in this repo) — "development" here means manifest/config changes only
- All cluster-specific configuration lives in `apps/media/radarr/helmrelease.yml`; in-app settings live in the persisted config PVC

### Change Management
- Image tag bumps are handled by Flux image automation markers (`$imagepolicy`) — manual tag edits in Git will be overwritten by automation unless the policy is paused
- Manifest changes (resources, ingress, volumes) go through the normal GitOps PR/commit flow for this repo

### Documentation Standards
- **API Documentation**: see `api-documentation.md` for the REST v3 reference
- **User Guide**: see `user-guide.md` for day-to-day usage
- **Deployment Guide**: see `deployment-guide.md` for install/upgrade/recovery procedures

This architecture provides a lightweight, self-contained movie automation component that integrates with the broader Prowlarr/FlareSolverr/Sonarr media stack while keeping its own state isolated to a single config volume.
