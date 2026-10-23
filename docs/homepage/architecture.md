---
title: Homepage Architecture Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [architecture, documentation, dashboard, kubernetes]
---

# Homepage Architecture Guide

## Overview

Homepage ([gethomepage.dev](https://gethomepage.dev)) is the central landing page for the Conder Home homelab. It is a single stateless Next.js application that renders a static-ish dashboard of service links, live status widgets, bookmarks, and system information. Every other service in the cluster (media stack, networking tools, infrastructure, monitoring) is surfaced here as one tile, so Homepage acts as the discovery/navigation hub for the whole homelab rather than performing any business logic of its own.

In this cluster it runs as a single-replica Deployment in the `tools` namespace, templated via the `bjw-s` `app-template` Helm chart and managed by Flux (HelmRelease `homepage`). All of its behavior is driven entirely by YAML files mounted from ConfigMaps — there is no database and no persistent application state beyond those config files.

## System Architecture

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    tools namespace (k3s)                      │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │             homepage Deployment (1 replica)           │    │
│  │  ┌──────────────────────────────────────────────┐     │    │
│  │  │   ghcr.io/gethomepage/homepage:latest         │     │    │
│  │  │   Next.js server (port 3000)                  │     │    │
│  │  │   - server-side widget proxy/fetcher          │     │    │
│  │  │   - renders dashboard UI                       │     │    │
│  │  └──────────────────────────────────────────────┘     │    │
│  │  Config mounted read-only from ConfigMaps:            │    │
│  │   services.yaml · settings.yaml · widgets.yaml        │    │
│  │   bookmarks.yaml · docker.yaml · kubernetes.yaml       │    │
│  │   proxmox.yaml                                         │    │
│  │  emptyDir: /app/config/logs                            │    │
│  └──────────────────────────────────────────────────────┘    │
│                            │                                  │
│                      Service (ClusterIP, :3000)                │
│                            │                                  │
│                 Traefik Ingress (homepage.conderhome.xyz)       │
└──────────────────────────────────────────────────────────────┘
                             │
                             ▼
        Outbound HTTPS/HTTP calls to every linked service's
        API (Plex, Sonarr, Radarr, Pi-hole, Proxmox, Grafana,
        TrueNAS, Immich, Home Assistant, Hermes, etc.) for
        live widget data — Homepage never stores this data.
```

### Core Components

#### 1. Application Container

**Homepage (Next.js) server**
- Single container, image `ghcr.io/gethomepage/homepage:latest`, `imagePullPolicy: Always`
- Listens on port 3000 internally (exposed via Service `http`)
- Renders the dashboard HTML/CSS and runs server-side API calls on behalf of the browser to each widget's backend (avoids CORS issues and keeps API keys off the client)
- No database, no persistent volumes for data — only a `logs` `emptyDir` for runtime logs and ConfigMap mounts for configuration

#### 2. Configuration Layer (ConfigMaps)

Homepage has **no admin UI for configuration** — everything is defined in seven YAML files, each backed by its own ConfigMap and mounted as a single file via `subPath`:

| File | ConfigMap | Purpose |
|------|-----------|---------|
| `services.yaml` | `homepage-services` | The actual dashboard content: groups, service tiles, links, icons, and live-status widgets. This is the file edited most often. |
| `settings.yaml` | `homepage-settings` | Global dashboard settings: title, background image, card blur, layout, weather provider keys. |
| `widgets.yaml` | `homepage-widgets` | "Info" widgets shown above the service tiles (resource usage, search bar, weather). |
| `bookmarks.yaml` | `homepage-bookmarks` | Simple bookmark links grouped separately from services. |
| `docker.yaml` | `homepage-docker` | Optional Docker socket/host integration for container-based widgets (currently commented out — unused, cluster is k3s not plain Docker). |
| `kubernetes.yaml` | `homepage-kubernetes` | Optional native Kubernetes integration config (currently a placeholder comment — unused). |
| `proxmox.yaml` | `homepage-proxmox` | Optional Proxmox cluster-wide widget config (currently a placeholder comment; per-service Proxmox widget in `services.yaml` is used instead). |

All seven ConfigMaps live in `apps/tools/homepage/configmaps/` and are aggregated by a local `kustomization.yaml`, which is itself included by the app's top-level `kustomization.yaml` alongside `helmrelease.yml`. There is also a legacy, unused `configmap.yml` (`homepage-config`) left in the directory from an earlier iteration — it is **not** referenced by the HelmRelease's `persistence` block and has no effect on the running app.

#### 3. Widget Engine

Each service entry in `services.yaml` can optionally declare a `widget:` block. Widgets fall into two families:

- **Native/integration widgets**: Homepage ships built-in adapters for dozens of self-hosted apps (Plex, Sonarr, Radarr, Prowlarr, qBittorrent, TrueNAS, Pi-hole, Unifi, Omada, NPM, Immich, Proxmox, Dockhand, Grafana, Home Assistant, etc.). Each adapter knows the target app's API shape and just needs `url` + credentials/API key.
- **`customapi` widget**: A generic adapter that calls any JSON HTTP endpoint and maps response fields to displayed labels via a `mappings` array — used here for the in-house `Hermes` service, which has no dedicated Homepage integration.

Widgets are fetched **server-side** by the Homepage container on every dashboard load/refresh, so the browser never talks directly to Grafana, Pi-hole, etc. — it only talks to Homepage.

#### 4. Networking / Ingress

- `Service` `app`: ClusterIP exposing container port 3000 as `http`
- `Ingress` `app`: Traefik `IngressClass`, host `homepage.conderhome.xyz`, TLS via `homepage-tls` secret (cert-manager issued)
- `HOMEPAGE_ALLOWED_HOSTS` env var is set to `homepage.conderhome.xyz` — Homepage validates the `Host` header against this and will reject requests presenting an unexpected host

### Data Flow Architecture

#### Dashboard Render / Widget Refresh

```
1. Browser requests https://homepage.conderhome.xyz
     ↓
2. Traefik Ingress → Service → homepage Pod (port 3000)
     ↓
3. Next.js server reads services.yaml / settings.yaml / widgets.yaml / bookmarks.yaml from disk (mounted ConfigMaps)
     ↓
4. For each service with a `widget:` block, server issues an outbound
   HTTP(S) request to that service's API using the configured url/credentials
     ↓
5. Widget responses mapped to display fields → merged into page data
     ↓
6. HTML/JSON returned to browser; widgets auto-poll on an interval for live updates
```

#### Config Change Flow (GitOps)

```
1. Developer edits a YAML file under apps/tools/homepage/configmaps/
     ↓
2. Commit + push to the infra-k8s repo
     ↓
3. Flux source-controller detects the Git change
     ↓
4. Flux kustomize-controller reconciles the ConfigMap object(s)
     ↓
5. Kubernetes updates the ConfigMap; kubelet syncs the mounted file in the Pod (eventual, typically within ~1 minute)
     ↓
6. Homepage re-reads YAML on next dashboard request — most settings apply live,
   no pod restart required (a manual rollout restart can be used to force it)
```

### Security Architecture

#### Authentication & Authorization
- Homepage itself has **no built-in login** in this deployment — anyone who can reach `homepage.conderhome.xyz` can view the dashboard and trigger widget calls. Exposure is limited to the homelab's internal network/VPN and Traefik ingress boundary.
- Per-widget credentials (usernames/passwords/API keys for Plex, TrueNAS, Unifi, Pi-hole, NPM, Omada, Dockhand, Grafana, Proxmox, Immich, Home Assistant) are stored in plaintext inside `services.yaml` today — this is the main secret-management weak point of the current setup (see Deployment Guide → Security Considerations).
- `HOMEPAGE_ALLOWED_HOSTS` provides basic host-header validation to reduce DNS-rebinding/host-spoofing risk.

#### Data Security
- No persistent data store — the only state is the YAML config and ephemeral logs (`emptyDir`, lost on pod restart).
- All outbound widget calls happen server-side, so third-party API keys are never shipped to the browser.
- TLS termination for the dashboard itself is handled by Traefik/cert-manager at the ingress layer.

### Deployment Architecture

#### Containerization
- Single official image `ghcr.io/gethomepage/homepage:latest`, pulled with `Always` policy — the cluster always runs upstream's newest build, there is no pinned version.
- No sidecars; `app-template` chart generates one container in one pod.

#### Orchestration
- **Flux HelmRelease** `homepage` (namespace `tools`) using the shared `bjw-s/app-template` chart (`version: "3.x"`), reconciled hourly (`interval: 1h`) or on Git push.
- **Deployment strategy**: `Recreate` (observed live) — the single pod is fully terminated before the replacement starts, so there is a brief dashboard outage on every config-triggered redeploy or image pull.
- **Replicas**: 1 — Homepage is not horizontally scaled; it's a lightweight, non-critical-path app.

#### Resource Management
- No CPU/memory requests or limits are currently set (`resources: {}` in the live Deployment) — Homepage runs unconstrained/best-effort on its node.
- No PersistentVolumeClaims — only ConfigMap mounts and one `emptyDir` for logs.

### Integration Points

#### External Services (all consumed as outbound widget targets)
- **Media**: Plex, TrueNAS, Radarr, Sonarr, Prowlarr, qBittorrent, ReadMeABook
- **Network**: Unifi controller, MySpeed, Pi-hole (x2), Omada controller
- **Services**: Termix, NGINX Proxy Manager, Immich, Vaultwarden, Actual Budget, NexusTale
- **Infrastructure**: Authentik, Proxmox, Dockhand, Grafana, Hermes (via `customapi`), Home Assistant, internal docs site

#### File System Integration
- `/app/config/services.yaml`, `settings.yaml`, `widgets.yaml`, `bookmarks.yaml`, `docker.yaml`, `kubernetes.yaml`, `proxmox.yaml` — each mounted individually via `subPath` from its own ConfigMap, so updating one file doesn't disturb the others.
- `/app/config/logs` — writable `emptyDir`, non-persistent application logs.

### Monitoring & Observability

#### Health Monitoring
- No explicit `livenessProbe`/`readinessProbe` is defined on the live Deployment — Kubernetes relies on container-process liveness only; a hung-but-running Next.js process would not be auto-restarted.
- Service reachability is effectively monitored indirectly: a broken widget renders an error/blank card on the dashboard itself, which is the primary operator-visible health signal.

#### Logging
- Container stdout/stderr (viewable via `kubectl logs`) plus the `emptyDir`-backed `/app/config/logs` directory the chart wires up.

### Performance Considerations

- Single replica, `Recreate` strategy — expect a short gap in dashboard availability during image updates or ConfigMap-triggered rollouts.
- Widget calls happen synchronously server-side per dashboard load; a slow or unreachable downstream service (e.g. Proxmox over the WAN) can slow down page render or leave that tile showing stale/blank data until the next poll.
- No caching layer in front of Homepage; it's lightweight enough that this is a non-issue at single-household scale.

## Technology Stack

- **Application**: Homepage (gethomepage.dev), Next.js-based, Node.js runtime, image `ghcr.io/gethomepage/homepage:latest`
- **Orchestration**: k3s, Flux v2 (`helm.toolkit.fluxcd.io/v2` HelmRelease), Kustomize
- **Helm chart**: `bjw-s/app-template` v3.x (generic templated chart used across this homelab, not Homepage-specific)
- **Ingress**: Traefik `IngressClass`, cert-manager-issued TLS
- **Config delivery**: Kubernetes ConfigMaps mounted as individual files via `subPath`

## Development Guidelines

### Code Structure (repo layout)
```
apps/tools/homepage/
├── helmrelease.yml           # HelmRelease: image, service, ingress, persistence mounts
├── configmap.yml             # legacy/unused "homepage-config" ConfigMap
├── kustomization.yaml        # includes configmap.yml, helmrelease.yml, configmaps/
└── configmaps/
    ├── kustomization.yaml    # includes the 7 config YAMLs below
    ├── services.yaml         # dashboard content — edited most often
    ├── settings.yaml         # global dashboard settings
    ├── widgets.yaml          # info widgets (resources/search/weather)
    ├── bookmarks.yaml        # bookmark links
    ├── docker.yaml           # unused Docker integration (commented out)
    ├── kubernetes.yaml       # unused k8s integration (placeholder)
    └── proxmox.yaml          # unused cluster-wide Proxmox integration (placeholder)
```

### Documentation Standards
- Config changes are GitOps-only: edit the YAML under `apps/tools/homepage/configmaps/`, commit, push, let Flux reconcile. Direct `kubectl edit` on the live ConfigMap will be reverted on the next reconcile.
- See the Deployment Guide for step-by-step instructions on adding a new service tile.

This architecture keeps Homepage intentionally simple: a single stateless container whose entire behavior is config-as-code, acting purely as a navigation and live-status aggregation layer over every other service in the homelab.
