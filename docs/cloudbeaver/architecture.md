---
title: CloudBeaver Architecture Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [architecture, documentation, database, kubernetes]
---

# CloudBeaver Architecture Guide

## Overview

CloudBeaver is a self-hosted, web-based database administration tool deployed in the `tools` namespace of the homelab k3s cluster. It provides a single browser UI for connecting to, browsing, and querying the various relational databases that live across the cluster — the shared `shared-postgres` CNPG cluster, `grafana-postgres`, a standalone MySQL deployment, and app-specific Postgres instances such as readmeabook's. CloudBeaver is a thin, stateless-per-request UI layer in front of those databases; it is not itself a database and holds no application data of its own beyond connection metadata and session workspace files.

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  CloudBeaver Pod (tools ns)                 │
│                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────┐ │
│  │  Java Backend    │  │  GWT Web UI    │  │  Session   │ │
│  │  (CE Server)     │  │  (Browser)     │  │  Manager   │ │
│  └─────────────────┘  └─────────────────┘  └────────────┘ │
│                                                             │
│  ┌─────────────────┐  ┌─────────────────┐                   │
│  │  JDBC Driver    │  │  Workspace     │                   │
│  │  Registry       │  │  (PVC-backed) │                   │
│  └─────────────────┘  └─────────────────┘                   │
└───────────────┬─────────────────┬───────────────────────────┘
                │ JDBC             │ JDBC
                ▼                 ▼
    ┌─────────────────────┐   ┌─────────────────────┐
    │  shared-postgres     │   │  grafana-postgres   │
    │  (CNPG cluster)      │   │  (CNPG cluster)     │
    └─────────────────────┘   └─────────────────────┘
                │ JDBC                      │ JDBC
                ▼                           ▼
    ┌─────────────────────┐   ┌─────────────────────┐
    │  mysql deployment    │   │  app-specific PG     │
    │                      │   │  (e.g. readmeabook)  │
    └─────────────────────┘   └─────────────────────┘
```

### Core Components

#### 1. Backend Service

**CloudBeaver Server (Community Edition)**
- Single Java process (`dbeaver/cloudbeaver` image) serving both the HTTP API and the compiled web client
- Handles authentication, session lifecycle, connection management, and SQL execution
- Bundles JDBC drivers for Postgres, MySQL, and other common engines so no external driver install is needed
- Admin bootstrap is driven by `CB_ADMIN_NAME` / `CB_ADMIN_PASSWORD` environment variables sourced from a Kubernetes Secret

**Workspace / Session Store**
- CloudBeaver persists its configuration, registered connections, user settings, and driver cache under `/opt/cloudbeaver/workspace`
- This directory is backed by a PersistentVolumeClaim (`cloudbeaver-data`, 1Gi, ReadWriteOnce, NFS storage class) so connection definitions and credentials survive pod restarts/rescheduling
- No external database is used for CloudBeaver's own metadata — it is self-contained on the PVC

#### 2. Frontend Application

**GWT-Compiled Web UI**
- Served directly by the same backend process (no separate frontend container)
- Browser-based SQL editor, result-set grid, ER diagram viewer, and connection/administration panels
- Communicates with the backend over its own internal REST/WebSocket API (see api-documentation.md)

#### 3. Database Connectivity Layer

**JDBC Driver Registry**
- Built-in drivers for PostgreSQL, MySQL, and other RDBMS engines
- Each registered connection stores host, port, database name, and credentials (encrypted at rest in the workspace)
- Connections are made directly from the CloudBeaver pod to target database Services over the cluster network — no separate proxy layer

**Target Databases in this Cluster**
- `shared-postgres` — CloudNativePG (CNPG) cluster used by multiple tools
- `grafana-postgres` — CNPG cluster backing Grafana's own metadata
- MySQL deployment — standalone MySQL instance for apps requiring it
- App-specific Postgres instances — e.g. readmeabook's dedicated database

### Data Flow Architecture

#### Query Execution Flow

```
1. Browser (SQL Editor) → 2. CloudBeaver Backend (parse/validate)
        ↓
3. JDBC Connection Pool → 4. Target Database (Postgres/MySQL)
        ↓
5. Result Set → 6. Backend (serialize) → 7. Browser (render grid)
```

1. **User writes SQL**: In the browser-based SQL editor against a selected connection
2. **Backend dispatch**: CloudBeaver backend opens/reuses a JDBC connection to the target database Service
3. **Execution**: Query runs on the target database (shared-postgres, grafana-postgres, mysql, or an app DB)
4. **Result streaming**: Rows are streamed back to the backend and paginated for the UI grid
5. **Session state**: Open connections, cursors, and transaction state are tracked per user session in memory and on the workspace volume

### Network Architecture

- **Ingress**: Traefik ingress at `cloudbeaver.conderhome.xyz`, routing to the `app` Service port
- **Service**: ClusterIP Service `cloudbeaver` in namespace `tools`, port `8978` → container port `8978`
- **Egress**: Pod reaches database Services (shared-postgres, grafana-postgres, mysql, app DBs) over the cluster's internal network/DNS — no external database endpoints required
- **No sidecars**: Single-container pod; no DRM, conversion, or queue services as seen in other apps in this cluster

### Security Architecture

#### Authentication & Authorization
- **Admin Bootstrap**: `CB_ADMIN_NAME` / `CB_ADMIN_PASSWORD` injected from the `cloudbeaver-secrets` Kubernetes Secret at container start
- **Per-user accounts**: Additional CloudBeaver users/roles can be created inside the admin UI once bootstrapped
- **Connection credentials**: Stored encrypted within the CloudBeaver workspace, scoped per connection
- **Ingress-level**: No additional auth proxy configured in the manifest; access control relies on CloudBeaver's own login plus network exposure via Traefik

#### Data Security
- **Secrets**: Admin credentials never appear in the HelmRelease manifest itself — sourced via `secretKeyRef`
- **Isolation**: CloudBeaver has no direct storage access to database data files; all access is mediated through JDBC and respects target database user permissions
- **Persistence**: Workspace PVC uses `ReadWriteOnce`, so only one pod can mount it at a time (consistent with `replicas: 1` and the `Recreate` deployment strategy)

### Deployment Architecture

#### Containerization
- **Single Image**: `dbeaver/cloudbeaver:25.0.1`
- **Deployment Strategy**: `Recreate` (not rolling) — appropriate given the single ReadWriteOnce workspace volume
- **Managed by**: Flux HelmRelease using the `bjw-s` `app-template` chart (version `3.x`)

#### Orchestration
- **Namespace**: `tools`
- **Controller**: Single `cloudbeaver` Deployment, 1 replica
- **Persistence**: `cloudbeaver-data` PVC (1Gi, NFS storage class) mounted at `/opt/cloudbeaver/workspace`
- **Networking**: ClusterIP Service on port 8978, exposed externally via Traefik Ingress at `cloudbeaver.conderhome.xyz`

#### Resource Management
- No explicit CPU/memory requests or limits are set in the current HelmRelease (`resources: {}`); the pod runs under cluster-default scheduling with whatever headroom the node has
- Storage is bounded by the 1Gi workspace PVC, which holds configuration and metadata only (not database data)

### Integration Points

#### Internal Database Services
- **shared-postgres (CNPG)**: General-purpose shared Postgres cluster
- **grafana-postgres (CNPG)**: Grafana's dedicated metadata store
- **mysql deployment**: Standalone MySQL instance
- **App-specific databases**: e.g. readmeabook's Postgres instance

#### External Services
- None. CloudBeaver does not call out to any third-party API; all of its functionality is local UI + JDBC to in-cluster databases.

### Monitoring & Observability

- **No dedicated health endpoint is configured** in the HelmRelease (no liveness/readiness probes defined); pod health is inferred from the standard Kubernetes Deployment `Available`/`Progressing` conditions
- **Logs**: Standard container stdout/stderr, retrievable via `kubectl logs -n tools -l app.kubernetes.io/name=cloudbeaver`
- **Status visibility**: `kubectl get pods -n tools -l app.kubernetes.io/name=cloudbeaver` and `kubectl describe pod` for scheduling/image/mount diagnostics

### Backup & Recovery

- **Workspace backup**: The `cloudbeaver-data` PVC (connection definitions, admin settings, driver cache) should be backed up like any other NFS-backed PVC in the cluster's volume backup routine
- **Database backup**: Not CloudBeaver's responsibility — backup of actual data lives with each target database (CNPG backup/restore for shared-postgres and grafana-postgres, standard MySQL dump procedures for the MySQL deployment)
- **Recreation**: Because CloudBeaver holds no primary data, a lost workspace volume only requires re-registering database connections, not data recovery

## Technology Stack

### Backend Technologies
- **Java**: CloudBeaver Community Edition server runtime
- **JDBC**: Database connectivity (PostgreSQL, MySQL drivers bundled)
- **GWT**: Compiled Java-to-JavaScript web client framework

### Infrastructure Technologies
- **Kubernetes (k3s)**: Container orchestration
- **Flux (HelmRelease)**: GitOps deployment via the `bjw-s/app-template` chart
- **Traefik**: Ingress controller / reverse proxy
- **NFS**: Persistent storage class backing the workspace PVC
- **CloudNativePG (CNPG)**: Operator providing the Postgres clusters CloudBeaver connects to

## Development Guidelines

### Manifest Structure
- **GitOps-managed**: All configuration lives in `apps/tools/cloudbeaver/helmrelease.yml`; changes are made via git commit + Flux reconciliation, not `kubectl edit`
- **Secrets separation**: Admin credentials are kept out of the manifest via `secretKeyRef` to `cloudbeaver-secrets`
- **Chart choice**: Uses the generic `bjw-s/app-template` chart pattern consistent with other simple single-container tools in this cluster (e.g. the audiofile and readmeabook style deployments)

This architecture gives the homelab a single, low-maintenance web UI for ad hoc SQL access across all cluster-hosted databases, without requiring any dedicated database client software on operator machines.
