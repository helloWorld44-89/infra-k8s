---
title: Prowlarr Architecture Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [architecture, documentation, media, kubernetes, indexers]
---

# Prowlarr Architecture Guide

## Overview

Prowlarr is a self-hosted indexer manager for the *arr media automation stack. It centralizes the configuration, health monitoring, and credential management of torrent/Usenet indexers, then syncs those indexer definitions outward to downstream applications — in this homelab, Radarr (movies) and Sonarr (TV). Rather than configuring indexers separately in each *arr app, Prowlarr acts as the single source of truth, pushing indexer config via each app's API whenever indexers are added, removed, or updated.

In this cluster, Prowlarr runs as a Kubernetes Deployment in the `media` namespace, deployed via Flux GitOps using the `bjw-s` `app-template` Helm chart, and depends on a co-located FlareSolverr instance to handle Cloudflare/CAPTCHA-protected indexers.

## System Architecture

### High-Level Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│                     media namespace (k3s)                         │
│                                                                     │
│  ┌─────────────────────┐        ┌────────────────────────────┐   │
│  │   Prowlarr Pod       │        │   FlareSolverr Pod         │   │
│  │   (linuxserver/      │◄──────►│   (ghcr.io/flaresolverr/   │   │
│  │    prowlarr:2.6.5)   │  HTTP  │    flaresolverr:latest)    │   │
│  │                      │ :8191  │                            │   │
│  │  ┌────────────────┐  │        │  ┌──────────────────────┐ │   │
│  │  │ .NET Backend   │  │        │  │ Headless Chromium +   │ │   │
│  │  │ + React UI     │  │        │  │ FlareSolverr proxy    │ │   │
│  │  └────────────────┘  │        │  │ (JS-rendering /       │ │   │
│  │  ┌────────────────┐  │        │  │  CAPTCHA bypass)      │ │   │
│  │  │ SQLite DB       │  │        │  └──────────────────────┘ │   │
│  │  │ (/config)       │  │        └────────────────────────────┘   │
│  │  └────────────────┘  │                                          │
│  └──────────┬───────────┘                                          │
│             │ Prowlarr API sync (apps/indexer config push)         │
│             ▼                                                      │
│  ┌─────────────────────┐        ┌────────────────────────────┐   │
│  │   Radarr Pod         │        │   Sonarr Pod               │   │
│  │   (movies)            │        │   (TV)                     │   │
│  └─────────────────────┘        └────────────────────────────┘   │
└───────────────────────────────────────────────────────────────────┘
                 ▲
                 │ Traefik Ingress (prowlarr.conderhome.xyz)
                 │ TLS via wildcard-conderhome-xyz-tls
                 │
            External / LAN clients
```

### Core Components

#### 1. Prowlarr Application Container

**Image**: `linuxserver/prowlarr` — a LinuxServer.io packaging of the upstream Prowlarr .NET application, auto-tracked by Flux's image-reflector/image-automation controllers via the `{"$imagepolicy": "flux-system:prowlarr:tag"}` marker.

- Written in .NET (same codebase lineage as Sonarr/Radarr/Lidarr), providing a web UI and REST API on port `9696`.
- Maintains its own SQLite database under `/config` for indexer definitions, sync profiles, application connections, history, and settings.
- Runs a background scheduler that periodically tests indexer health (via "RSS"/search test requests) and re-pushes indexer config to connected applications.

#### 2. Indexer Management Layer

- **Indexer Definitions**: Prowlarr ships with and auto-updates a large catalog of public and private indexer definitions (via the Indexer Definition Updater, pulling from the Prowlarr/Indexers community repo).
- **Indexer Proxies**: Optional per-indexer proxy configuration, including FlareSolverr, for indexers that present Cloudflare JS challenges or CAPTCHAs before serving search results.
- **Sync Profiles**: Define which indexer categories/tags are pushed to which downstream application, enabling, for example, movie-only indexers to sync to Radarr and TV-only indexers to sync to Sonarr.

#### 3. FlareSolverr Integration

FlareSolverr is deployed as a sibling workload in the same `media` namespace (`apps/media/flaresolverr/helmrelease.yml`), running `ghcr.io/flaresolverr/flaresolverr:latest`. It exposes a lightweight HTTP API on port `8191` backed by a headless Chromium browser that can solve Cloudflare's "I'm Under Attack" / JS-challenge pages and basic CAPTCHA challenges, returning the resolved page content (and cookies) to the caller.

**How Prowlarr uses it:**

- Several configured indexers front their search/download endpoints with Cloudflare protection. Prowlarr cannot complete a plain HTTP request against these without failing the JS challenge.
- Inside Prowlarr's UI (**Settings → Indexers → Indexer Proxies**), a FlareSolverr proxy entry is configured pointing at FlareSolverr's **in-cluster Service DNS name**, not its pod IP or an external hostname:

  ```
  http://flaresolverr.media.svc.cluster.local:8191
  ```

  (Short form `http://flaresolverr:8191` also resolves correctly for same-namespace pods via k3s's internal DNS, since both Prowlarr and FlareSolverr live in `media`.)

- Any indexer that needs challenge-bypassing has that indexer proxy attached in its settings. When Prowlarr searches or tests that indexer, it transparently routes the HTTP request through FlareSolverr first; FlareSolverr renders the page, solves the challenge, and returns the final response body/cookies back to Prowlarr, which then parses results as normal.
- This keeps challenge-solving logic (headless browser, JS execution) isolated to a single dedicated workload, so it can be scaled, restarted, or resource-constrained independently of Prowlarr itself, and so Radarr/Sonarr never need their own FlareSolverr proxy — Prowlarr is the only consumer.
- Dependency direction is one-way: Prowlarr calls FlareSolverr; FlareSolverr has no knowledge of or dependency on Prowlarr. There's no shared storage or state between them.

#### 4. Application Sync to Radarr / Sonarr

- Configured under **Settings → Apps** in Prowlarr, pointing at each app's in-cluster Service URL (e.g., `http://radarr.media.svc.cluster.local:7878`, `http://sonarr.media.svc.cluster.local:8989`) plus that app's API key.
- On a sync interval (and on indexer add/remove/edit), Prowlarr calls the target app's `/api/v3/indexer` endpoints to create/update/delete matching indexer entries, tagged by category compatibility (movie categories → Radarr, TV categories → Sonarr).
- This is a push model: Prowlarr owns indexer truth; Radarr/Sonarr are kept in sync automatically and should not have indexers hand-edited there, since the next sync would overwrite manual changes.

### Data Flow Architecture

#### Indexer Sync Pipeline

```
1. Indexer Added/Updated in Prowlarr UI
     ↓
2. (If proxy attached) Test search routed through FlareSolverr
     ↓
3. Indexer validated (capabilities, categories, auth)
     ↓
4. Sync Profile evaluated → matching Applications identified
     ↓
5. Prowlarr pushes indexer config via Radarr/Sonarr API
     ↓
6. Radarr/Sonarr store indexer, available for search/RSS
```

#### Search Request Flow (Cloudflare-protected indexer)

```
1. User/Radarr/Sonarr triggers a search in Prowlarr
2. Prowlarr builds the indexer's search request
3. Indexer has a FlareSolverr proxy attached → request routed to
   http://flaresolverr.media.svc.cluster.local:8191
4. FlareSolverr launches/reuses a headless Chromium session,
   loads the target URL, solves the Cloudflare challenge
5. FlareSolverr returns resolved HTML + cookies to Prowlarr
6. Prowlarr parses the response into normalized release results
7. Results returned to caller (UI, or Radarr/Sonarr via Prowlarr's
   indexer proxy endpoint)
```

### Kubernetes Deployment Topology

- **Namespace**: `media`
- **Workload**: `Deployment/prowlarr` (1 replica, `Recreate` strategy — appropriate since SQLite on a `ReadWriteOnce` volume cannot tolerate rolling/parallel pods)
- **Container**: single `app` container, image `linuxserver/prowlarr:2.6.5`
- **Service**: `prowlarr` ClusterIP, port `9696` → pod port `9696`
- **Ingress**: Traefik `IngressRoute`-style ingress at `prowlarr.conderhome.xyz`, TLS via the shared `wildcard-conderhome-xyz-tls` secret
- **Persistence**: single PVC `prowlarr-config` (`ReadWriteOnce`, 1Gi) mounted at `/config` — holds the SQLite database, logs, and backups
- **Sibling workload**: `Deployment/flaresolverr` in the same namespace, Service `flaresolverr` ClusterIP on port `8191`

### Security Architecture

- **Network boundary**: Prowlarr's UI/API is exposed externally only through the Traefik Ingress with TLS termination; FlareSolverr has **no Ingress** and is reachable only inside the cluster, which is intentional — it should never be internet-facing since it's purely a backend helper for Prowlarr.
- **Secrets**: Indexer credentials (API keys, cookies, login credentials for private trackers) are stored inside Prowlarr's own SQLite database under `/config`, not as Kubernetes Secrets — losing or resetting the PVC loses all indexer credentials.
- **No shared auth**: There is no API key, token, or mTLS exchanged between Prowlarr and FlareSolverr; FlareSolverr's HTTP API is unauthenticated by design, which is why it must remain cluster-internal only.
- **Pod security**: runs with the default (empty) `securityContext`, `default` ServiceAccount with `automountServiceAccountToken: true` — standard for this cluster's app-template-based workloads, not hardened beyond defaults.

### Monitoring & Observability

- **Readiness/liveness**: not explicitly configured in the current HelmRelease (relies on the app-template chart defaults / k3s pod status).
- **Logs**: container stdout/stderr, retrievable via `kubectl logs -n media -l app.kubernetes.io/name=prowlarr`.
- **Indexer health**: visible in the Prowlarr UI under **Indexers**, which flags failing/rate-limited indexers; FlareSolverr connectivity issues typically surface as indexer test failures referencing a proxy timeout or connection refused error.

### Technology Stack

- **Prowlarr**: .NET application (LinuxServer.io image), SQLite, React-based web UI, port 9696
- **FlareSolverr**: Python application wrapping headless Chromium (via Selenium/undetected-chromedriver equivalents), port 8191
- **Orchestration**: k3s, Flux CD v2 (`HelmRelease` + `HelmRepository` for the `bjw-s` `app-template` chart v3.x)
- **Ingress**: Traefik, cert-manager-issued wildcard TLS certificate
- **Storage**: Kubernetes `PersistentVolumeClaim` (ReadWriteOnce)

### Integration Points

- **Radarr** (movies) — receives indexer config pushes from Prowlarr via Radarr's API
- **Sonarr** (TV) — receives indexer config pushes from Prowlarr via Sonarr's API
- **FlareSolverr** — consumed by Prowlarr as an indexer proxy for Cloudflare/CAPTCHA-protected indexers, over `http://flaresolverr.media.svc.cluster.local:8191`
- **Flux CD** — manages the HelmRelease lifecycle and automated image tag updates via image policy annotations

This architecture keeps Prowlarr as the single indexer-management control plane for the homelab's media stack, with FlareSolverr as a narrowly-scoped, internally-reachable helper service that lets Prowlarr reach indexers that would otherwise be blocked by anti-bot challenges.

---

*This architecture guide documents the Prowlarr deployment in the conderhome homelab k3s cluster, managed via Flux GitOps.*
