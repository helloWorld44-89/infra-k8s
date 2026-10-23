---
title: MySQL Architecture Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [architecture, documentation, database, kubernetes]
---

# MySQL Architecture Guide

## Overview

MySQL is a shared, general-purpose relational database service running in the `tools` namespace of the homelab k3s cluster. It is deployed as a single-replica MySQL 8 instance via the `bjw-s` `app-template` Helm chart, managed by Flux through a `HelmRelease`. Unlike most workloads in this cluster, it has no end-user web UI — it is a plain database engine exposed only on the MySQL wire protocol (port 3306) for use by other in-cluster workloads and by administrators via CloudBeaver or the `mysql` CLI.

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     tools namespace                         │
│                                                             │
│  ┌─────────────────┐        ┌─────────────────┐            │
│  │  CloudBeaver     │------->│  MySQL Service  │            │
│  │  (DB admin UI)   │  3306  │  (ClusterIP)    │            │
│  └─────────────────┘        └────────┬────────┘            │
│                                       │                      │
│                                       ▼                      │
│                              ┌─────────────────┐            │
│                              │  mysql Pod       │            │
│                              │  (mysql:8 image) │            │
│                              │  - mysqld        │            │
│                              └────────┬────────┘            │
│                                       │                      │
│                                       ▼                      │
│                              ┌─────────────────┐            │
│                              │  PVC: mysql-data │            │
│                              │  (NFS, 1Gi, RWO) │            │
│                              │  /var/lib/mysql  │            │
│                              └─────────────────┘            │
└─────────────────────────────────────────────────────────────┘
```

### Core Components

#### 1. Database Engine

**MySQL 8 Container**
- Official `mysql:8` image, single `app` container inside a Deployment named `mysql`
- Runs as a standard `mysqld` server process, no custom entrypoint or sidecars
- Database name, root password, and application user/password are all injected from a Kubernetes `Secret` (`mysql-secrets`) via `secretKeyRef` — no credentials are stored in the HelmRelease manifest itself
- Deployment strategy is `Recreate` (not `RollingUpdate`), which is appropriate for a single-writer stateful database bound to one RWO volume

**Default Database**
- A single schema, `my_database`, is created automatically on first boot via `MYSQL_DATABASE`
- A non-root application user is provisioned via `MYSQL_USER` / `MYSQL_PASSWORD` and is granted access to that schema (standard `mysql:8` entrypoint behavior)
- The root account is also available via `MYSQL_ROOT_PASSWORD` for administrative tasks

#### 2. Networking

**ClusterIP Service**
- Service name: `mysql`, namespace: `tools`
- Exposes port `3306` (named `http` in the chart values — a chart convention, not an actual HTTP port; the protocol is the MySQL binary wire protocol)
- In-cluster DNS address: `mysql.tools.svc.cluster.local:3306`
- No Ingress is defined — MySQL is not reachable from outside the cluster network, and is not intended to be (databases should not be exposed to the public internet directly)

#### 3. Storage

**Persistent Volume**
- A single PVC, `mysql-data`, backed by the cluster's `nfs` StorageClass
- Access mode `ReadWriteOnce`, size `1Gi`
- Mounted at `/var/lib/mysql` inside the container — this is the full MySQL data directory (InnoDB tablespaces, binlogs if enabled, grant tables, etc.)
- Because storage is NFS-backed, only one pod can safely write to it at a time, which aligns with the single-replica, `Recreate`-strategy deployment model

#### 4. Administration Layer

**CloudBeaver**
- A separate HelmRelease (`apps/tools/cloudbeaver`) runs `dbeaver/cloudbeaver`, a web-based universal database client, in the same `tools` namespace
- CloudBeaver is exposed via Traefik Ingress at `https://cloudbeaver.conderhome.xyz` and authenticates with its own admin credentials (`cloudbeaver-secrets`)
- It is configured by an administrator (not declared in Git) with a connection pointing at `mysql.tools.svc.cluster.local:3306`, and is the primary way humans browse, query, and manage data in this MySQL instance without a `kubectl exec` or port-forward

### Data Flow

```
1. Workload/Admin connects → 2. MySQL wire protocol handshake → 3. Auth against mysql.user table
     ↓                              ↓                                   ↓
4. Query execution (InnoDB) → 5. Read/write to /var/lib/mysql (NFS PVC) → 6. Result set returned
```

### Consumers and Purpose

As of this writing, **no other HelmRelease in this repository declares `DB_HOST`/`MYSQL_HOST`/`DATABASE_URL` environment variables pointing at `mysql.tools.svc.cluster.local`** — a repo-wide grep across `apps/` turns up no app wired to this instance. The other database-dependent app in the cluster, Immich (`apps/media/immich`), uses its own dedicated `shared-postgres` PostgreSQL service instead.

In practice this MySQL deployment currently serves as:

- **A general-purpose/shared database utility** for the `tools` namespace — available for any future app, script, or manual workload that needs a MySQL-compatible backend without standing up its own database
- **An administrator-operated database**, reached and managed directly through **CloudBeaver** (web SQL client) or the **`mysql` CLI** via `kubectl exec`/port-forward, rather than being consumed by an automated application today

If a new application is wired to this database in the future, update this section with the consuming app, its database/schema, and connection env vars.

### Security Architecture

#### Authentication & Authorization
- MySQL's own native user/grant system (`mysql.user`, `mysql.db` tables) — root account plus one application user scoped to `my_database`
- All credentials are Kubernetes `Secret` objects (`mysql-secrets`), referenced by key, never inlined in plaintext manifests
- No TLS is configured on the MySQL listener — traffic relies on the cluster's internal pod network boundary rather than wire encryption

#### Network Security
- `ClusterIP` only — no Ingress, no NodePort, no LoadBalancer
- Only reachable from within the k3s cluster network (or via explicit `kubectl port-forward` for ad-hoc external access)

#### Data Security
- Single PVC on NFS; no automated backup job is defined in this HelmRelease (see Deployment Guide for manual backup/restore)
- `MYSQL_ROOT_PASSWORD` provides full administrative access — should be rotated via the `mysql-secrets` Secret if ever exposed

### Deployment Architecture

#### Orchestration
- **Flux `HelmRelease`**: `apps/tools/mysql/helmrelease.yml`, reconciled hourly (`interval: 1h`) from the `bjw-s` `app-template` chart (`version: "3.x"`)
- **Kustomization**: `apps/tools/mysql/kustomization.yaml` bundles the `HelmRelease` and `secrets.yml`
- **Namespace**: `tools`, alongside CloudBeaver, Homepage, docs-site, Sonarr, and Radarr

#### Resource Management
- Single replica, no resource requests/limits currently defined in the chart values
- Storage fixed at `1Gi` — should be monitored and expanded if the database grows beyond a small utility workload

### Monitoring & Observability

#### Health Monitoring
- No explicit liveness/readiness probes are defined in the HelmRelease; Kubernetes relies on default container restart behavior
- Pod and Deployment status can be checked with standard `kubectl get pods -n tools -l app.kubernetes.io/name=mysql` / `kubectl describe pod`

#### Logging
- Standard MySQL server logs are written to container stdout/stderr and retrievable via `kubectl logs`
- No external log aggregation or slow-query log shipping is configured

### Technology Stack

- **MySQL 8**: Database engine (`mysql:8` Docker image)
- **bjw-s app-template (Helm chart v3.x)**: Generic Kubernetes workload chart used across this cluster for most apps
- **Flux v2 (`helm.toolkit.fluxcd.io/v2`)**: GitOps reconciliation
- **NFS StorageClass**: Persistent volume backend
- **CloudBeaver (`dbeaver/cloudbeaver:25.0.1`)**: Web-based administration client

This architecture provides a lightweight, Git-managed MySQL instance intended as shared infrastructure for the homelab's `tools` namespace, administered through CloudBeaver and the standard `mysql` client rather than through any bespoke application API.
