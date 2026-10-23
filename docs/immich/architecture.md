---
title: Immich Architecture Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [architecture, documentation, photos, kubernetes]
---

# Immich Architecture Guide

## Overview

Immich is a self-hosted photo and video management platform providing automatic backup, organization, facial recognition, and media search for personal libraries. In this homelab it runs as a Flux-managed HelmRelease in the `media` namespace of the k3s cluster, and was migrated from a standalone Docker host (`192.168.5.40`) where it previously ran as a set of standalone containers.

## System Architecture

### High-Level Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│                     k3s Cluster — media namespace                 │
│                                                                   │
│  ┌──────────────────┐   ┌───────────────────────┐  ┌───────────┐ │
│  │  immich-server    │   │ immich-machine-learning│  │  immich-  │ │
│  │  (API + Web UI +   │◄──┤ (CLIP, face detection, │  │  valkey   │ │
│  │   job workers)     │   │  smart search models)  │  │ (cache +  │ │
│  └─────────┬─────────┘   └───────────┬────────────┘  │  queues)  │ │
│            │                          │               └─────┬─────┘ │
│            │ persistent connection     │ model cache PVC     │       │
│            │                          │                      │       │
└────────────┼──────────────────────────┼──────────────────────┼───────┘
             │                          │                      │
             ▼                          │                      ▼
┌────────────────────────────┐          │           (job/session cache,
│  databases namespace        │          │            shared with server)
│  shared-postgres (CNPG)     │          │
│  database: "immich"         │◄─────────┘
│  shared-postgres-rw svc     │
└────────────────────────────┘
             │
             ▼
┌────────────────────────────┐
│  immich-library-pvc (1Ti,   │
│  RWX) — original photos,    │
│  thumbnails, encoded video   │
└────────────────────────────┘
```

### Core Components

#### 1. `immich-server` Deployment

- The main application container (`ghcr.io/immich-app/immich-server`), exposing the REST API and serving the Web UI on container port `2283`.
- Also runs Immich's internal background job workers (thumbnail generation, metadata extraction, video transcoding, library watching, storage template migration) in-process — there is no separate worker Deployment in this chart's configuration.
- Backed by a `ClusterIP` Service (`immich-server`) and a Traefik `Ingress` for `immich.conderhome.xyz`.

#### 2. `immich-machine-learning` Deployment

- Runs Immich's ML service (`ghcr.io/immich-app/immich-machine-learning`), which serves CLIP-based smart search, facial recognition/clustering, and object detection models over an internal gRPC/HTTP API on port `3003`.
- `immich-server` calls into this service for ML-backed jobs; results (face/object metadata, CLIP embeddings) are written back to Postgres.
- Has its own persistence (`cache` volume enabled) so downloaded ONNX models survive pod restarts instead of re-downloading on every restart.

#### 3. `immich-valkey` Deployment

- A dedicated Valkey (Redis-compatible) instance (`valkey/valkey`), deployed via the chart's bundled `valkey.enabled: true` sub-chart.
- Used by `immich-server` as its job queue backend (BullMQ) and for short-lived caching — **not** shared with any other application in the cluster. This is a per-application cache/queue, distinct from the shared Postgres database described below.

#### 4. Database Layer — Shared CloudNative-PG Postgres

**This is the most important architectural deviation from most other apps in this cluster.**

- Immich does **not** run its own dedicated Postgres instance. Instead it connects to the cluster-wide **shared-postgres** CloudNative-PG cluster that lives in the `databases` namespace.
- Connection details (from the HelmRelease env):
  - `DB_HOSTNAME`: `shared-postgres-rw.databases.svc.cluster.local`
  - `DB_PORT`: `5432`
  - `DB_DATABASE_NAME`: `immich`
  - Credentials are injected via the `immich-db-credentials` Kubernetes Secret (`username`/`password` keys), not inline in the HelmRelease.
- Immich requires the `pgvecto.rs`/`vectors` Postgres extension for storing and querying CLIP embeddings (smart search) and face embeddings — this must be present on the shared-postgres cluster's image for Immich's ML features to function.
- Because the database is shared infrastructure, Immich's schema lives alongside other applications' databases on the same CNPG cluster; backup/restore of `shared-postgres` covers Immich's data too, but any maintenance on `shared-postgres` (failover, major version upgrades) directly affects Immich availability.

#### 5. Storage

- `immich-library-pvc`: a 1Ti `ReadWriteMany` PersistentVolumeClaim mounted into `immich-server`, holding the original uploaded photos/videos, generated thumbnails, encoded videos, and profile images. RWX access mode allows this to be served from NFS-backed storage so the volume could, in principle, be mounted by multiple pods (e.g. if the server deployment were scaled or a CLI job pod needed direct access).
- `immich-machine-learning` has a separate `cache` PVC for its downloaded model weights, decoupled from the photo library volume.

### Networking

- **Ingress**: Traefik `IngressRoute`/`Ingress` for `immich.conderhome.xyz`, TLS terminated with the wildcard `wildcard-conderhome-xyz-tls` certificate (cert-manager managed).
- **Internal services**: `immich-server` (port 2283), `immich-machine-learning` (port 3003), and `immich-valkey` (port 6379) are all `ClusterIP` services, reachable only inside the cluster except through the Ingress.
- All inter-component traffic (server → machine-learning, server → valkey, server → shared-postgres) stays on the cluster's internal pod network; nothing but HTTPS on port 443 is exposed externally.

### Data Flow

#### Upload / Backup Pipeline

```
1. Mobile app / Web UI upload → 2. immich-server receives asset
     ↓
3. Asset written to immich-library-pvc → 4. Metadata extracted (EXIF, duration, etc.)
     ↓
5. Job enqueued in Valkey (thumbnail, encode, ML) → 6. Thumbnail generation
     ↓
7. Machine-learning service: face/object detection, CLIP embedding → 8. Results written to shared-postgres (immich DB)
     ↓
9. Asset now searchable, visible in timeline, albums, map view
```

#### Smart Search Pipeline

1. User submits a natural-language or image-based search query via the API/UI.
2. `immich-server` requests a CLIP embedding for the query from `immich-machine-learning`.
3. The embedding is compared against stored asset embeddings using the `vectors` Postgres extension (`shared-postgres`, `immich` database) via a similarity query.
4. Matching assets are returned, ranked by vector distance.

### Migration History (Docker → k3s)

- **Previous deployment**: Immich ran as standalone Docker containers on host `192.168.5.40` — `immich_server`, `immich_postgres`, `immich_redis`, and `immich_machine_learning`. Those containers are stopped on that host but intentionally not deleted, pending final confidence in the k3s deployment.
- **Current deployment**: Flux-managed HelmRelease in the k3s `media` namespace, using the **shared** `shared-postgres` CNPG cluster instead of a dedicated `immich_postgres` container, and a cluster-native `immich-valkey` Deployment instead of the old `immich_redis` container.
- **Migration verification**: row counts in the k3s `immich` Postgres database were compared against the stopped Docker `immich_postgres` container's data. The k3s database has **more** assets than the old Docker database, confirming the k3s deployment is the live, authoritative copy (continuing to receive new uploads) rather than a stale snapshot taken at migration time.

### Security Architecture

- **Authentication**: Immich's own built-in user/password auth, with optional OAuth (`oauthButtonText`/`oauthAccountManagementUrl` are configurable via server config, currently using Immich's native login).
- **TLS**: All external traffic terminates TLS at the Traefik ingress using the shared wildcard certificate; internal cluster traffic is unencrypted ClusterIP.
- **Secrets**: Postgres credentials are stored in the `immich-db-credentials` Secret and injected as environment variables — never committed in plaintext to the HelmRelease.
- **Network boundary**: machine-learning and valkey services have no ingress of their own; they are only reachable from other pods in the `media` namespace (or wherever NetworkPolicies allow).

### Deployment Architecture

- **Orchestration**: Flux `HelmRelease` (`apps/media/immich/helmrelease.yaml`), pulling the official `immich` Helm chart from the `immich-charts` HelmRepository, pinned to chart version `0.13.x` and app image tag `v3.2.2`.
- **Reconciliation**: Flux reconciles the HelmRelease every `1h`, or immediately on a Git push that touches the manifest.
- **Namespace**: `media`, shared with other media-stack applications; `app.kubernetes.io/instance=immich` selects all three Immich-related pods/deployments for cluster-wide queries.
- **Replicas**: all three components currently run as single replicas (no HPA configured), consistent with a single-tenant homelab deployment.

### Monitoring & Observability

- Pod health is tracked via standard k3s/kubectl tooling (`kubectl get pods -n media`); the Helm chart does not currently define custom Prometheus scrape annotations in this deployment's values.
- Immich exposes a server version/info endpoint (`/api/server/version`) useful for confirming the running release after an upgrade.
- Logs are retrieved per-pod via `kubectl logs` (see the deployment guide for exact commands).

### Backup & Recovery Considerations

- **Database**: handled as part of the shared `shared-postgres` CNPG cluster's backup strategy (the `immich` database is one of several logical databases on that cluster) — there is no Immich-specific database backup job.
- **Media library**: the `immich-library-pvc` (1Ti RWX volume) is the single source of truth for original assets and must be included in any storage-level backup/snapshot strategy independently of the database.
- **Model cache**: the machine-learning `cache` PVC is disposable — if lost, models are simply re-downloaded on next startup.

This architecture reflects a deliberate homelab pattern: application compute (server, ML, cache) is deployed per-app, while stateful infrastructure (Postgres) is centralized on shared, more carefully operated infrastructure — reducing the number of database clusters to manage while keeping Immich's own queue/cache and media storage dedicated to the app.
