---
title: Immich Deployment Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [deployment, kubernetes, flux, infrastructure]
---

# Immich Deployment Guide

## Overview

Immich runs in this homelab as a Flux-managed `HelmRelease` on the k3s cluster, in the `media` namespace. This guide covers the actual deployed configuration, how to operate it day-to-day, and the history of its migration off the old Docker host.

## Current Deployment

### Manifest Location

```
apps/media/immich/helmrelease.yaml
```

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: immich
  namespace: media
spec:
  interval: 1h
  chart:
    spec:
      chart: immich
      version: "0.13.x"
      sourceRef:
        kind: HelmRepository
        name: immich-charts
        namespace: flux-system
  values:
    controllers:
      main:
        containers:
          main:
            image:
              tag: v3.2.2
            env:
              DB_HOSTNAME: shared-postgres-rw.databases.svc.cluster.local
              DB_PORT: "5432"
              DB_DATABASE_NAME: immich
              DB_USERNAME:
                valueFrom:
                  secretKeyRef:
                    name: immich-db-credentials
                    key: username
              DB_PASSWORD:
                valueFrom:
                  secretKeyRef:
                    name: immich-db-credentials
                    key: password

    immich:
      persistence:
        library:
          existingClaim: immich-library-pvc

    valkey:
      enabled: true

    server:
      ingress:
        main:
          enabled: true
          className: traefik
          hosts:
            - host: immich.conderhome.xyz
              paths:
                - path: /
                  pathType: Prefix
                  service:
                    identifier: main
                    port: http
          tls:
            - hosts:
                - immich.conderhome.xyz
              secretName: wildcard-conderhome-xyz-tls

    machine-learning:
      enabled: true
      persistence:
        cache:
          enabled: true
```

### Live Resources (as deployed)

| Resource | Name | Notes |
|----------|------|-------|
| Deployment | `immich-server` | API + Web UI + job workers, image `ghcr.io/immich-app/immich-server:v3.2.2` |
| Deployment | `immich-machine-learning` | CLIP/face/object detection models, image `ghcr.io/immich-app/immich-machine-learning:v3.2.2` |
| Deployment | `immich-valkey` | Job queue + cache, image `valkey/valkey:9.1.2-alpine` |
| Service | `immich-server` | ClusterIP, port `2283` |
| Service | `immich-machine-learning` | ClusterIP, port `3003` |
| Service | `immich-valkey` | ClusterIP, port `6379` |
| PVC | `immich-library-pvc` | `1Ti`, `RWX` — original photo/video library |
| Ingress | `immich-server` | Traefik, host `immich.conderhome.xyz`, TLS via `wildcard-conderhome-xyz-tls` |
| Secret | `immich-db-credentials` | `username`/`password` for shared-postgres `immich` database |

All three deployments currently run a single replica each; there is no HPA configured.

## Prerequisites

### Cluster Dependencies

- **k3s cluster** with Flux CD installed and reconciling `flux-system`
- **CloudNative-PG `shared-postgres` cluster** already running in the `databases` namespace, with a logical database named `immich` and the `pgvecto.rs`/vector extension available (required for smart search and face embeddings)
- **Traefik** ingress controller and **cert-manager** issuing the `wildcard-conderhome-xyz-tls` certificate
- **Storage class** capable of provisioning a 1Ti `ReadWriteMany` volume (NFS-backed in this cluster) for `immich-library-pvc`

### HelmRepository

The `immich-charts` `HelmRepository` must exist in `flux-system` pointing at the official Immich Helm chart source before this `HelmRelease` can reconcile.

## Deploying / Updating

### Initial Deployment

1. Ensure `shared-postgres` has a database named `immich` and a corresponding role with credentials stored in the `immich-db-credentials` Secret in the `media` namespace:
   ```bash
   ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl get secret immich-db-credentials -n media"
   ```
2. Ensure `immich-library-pvc` exists and is bound:
   ```bash
   ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl get pvc -n media immich-library-pvc"
   ```
3. Commit `apps/media/immich/helmrelease.yaml` to the Git repo that Flux watches; Flux will reconcile within the `1h` interval, or immediately on the next `flux reconcile`.
4. Force an immediate reconcile if needed:
   ```bash
   ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config flux reconcile helmrelease immich -n media"
   ```

### Upgrading Immich

1. Bump `image.tag` (and/or chart `version`) in `apps/media/immich/helmrelease.yaml`.
2. Commit and push; Flux reconciles automatically, or trigger manually as above.
3. Watch the rollout:
   ```bash
   ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl rollout status deploy/immich-server -n media"
   ```
4. Confirm the version matches post-upgrade:
   ```bash
   ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl exec -n media deploy/immich-server -- curl -s http://localhost:2283/api/server/version"
   ```

## Operations

### Checking Status

```bash
# Pods
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl get pods -n media | grep immich"

# Deployments / HelmRelease status
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl get deploy -n media -l app.kubernetes.io/instance=immich"
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl get helmrelease immich -n media"

# Ingress / routing
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl get ingress -n media | grep immich"
```

### Logs

```bash
# Server logs
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl logs -n media deploy/immich-server -f"

# Machine learning service logs
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl logs -n media deploy/immich-machine-learning -f"

# Valkey logs
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl logs -n media deploy/immich-valkey -f"
```

### Health Checks

```bash
# From inside the cluster
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl exec -n media deploy/immich-server -- curl -s http://localhost:2283/api/server/version"
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl exec -n media deploy/immich-server -- curl -s http://localhost:2283/api/server/config"

# From outside
curl -s https://immich.conderhome.xyz/api/server/version
```

### Restarting Components

```bash
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl rollout restart deploy/immich-server -n media"
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl rollout restart deploy/immich-machine-learning -n media"
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl rollout restart deploy/immich-valkey -n media"
```

## Database Operations (Shared Postgres)

Immich uses the **shared** `shared-postgres` CloudNative-PG cluster in the `databases` namespace — not a dedicated Postgres instance. This has operational implications:

- **Do not** treat Postgres maintenance on `shared-postgres` as Immich-specific; a failover or restart affects every application sharing that cluster.
- **Connecting directly** to inspect the `immich` database:
  ```bash
  ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl exec -it -n databases shared-postgres-1 -- psql -U postgres -d immich"
  ```
- **Row-count sanity check** (used during migration verification, useful again after any restore):
  ```sql
  SELECT count(*) FROM assets;
  ```
- Backups of the `immich` database are covered by whatever backup strategy is configured for `shared-postgres` as a whole (see that cluster's own documentation) — there is no separate Immich-specific pg_dump job in this deployment.

## Migration History: Docker Host → k3s

### Background

Immich previously ran as standalone Docker containers on host `192.168.5.40`:

- `immich_server`
- `immich_postgres` (dedicated, not shared)
- `immich_redis`
- `immich_machine_learning`

All four containers are **stopped but not deleted** on `192.168.5.40`, retained as a fallback until the k3s deployment's long-term stability is fully confirmed.

### What Changed

| Aspect | Old (Docker host) | New (k3s) |
|--------|--------------------|-----------|
| Postgres | Dedicated `immich_postgres` container | Shared `shared-postgres` CNPG cluster, `immich` database |
| Redis/Valkey | `immich_redis` container | Dedicated `immich-valkey` Deployment (still per-app, not shared) |
| Orchestration | Manual `docker run`/compose on `192.168.5.40` | Flux `HelmRelease`, GitOps-managed |
| Ingress/TLS | Host-level reverse proxy (if any) | Traefik Ingress + cert-manager wildcard cert |
| Storage | Local disk on `192.168.5.40` | `immich-library-pvc` (1Ti RWX, NFS-backed) |

### Verifying the Migration Was Successful

The migration was confirmed by comparing asset row counts between the two Postgres databases:

```bash
# Old Docker Postgres (while container temporarily started for comparison, or via retained data)
docker exec immich_postgres psql -U postgres -d immich -c "SELECT count(*) FROM assets;"

# New k3s shared-postgres
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl exec -it -n databases shared-postgres-1 -- psql -U postgres -d immich -c 'SELECT count(*) FROM assets;'"
```

**Result**: the k3s database had a **higher** asset count than the old Docker database, confirming:

1. The k3s deployment is the live, authoritative copy of the library (continuing to accept new uploads from mobile/web clients).
2. It is not a stale snapshot frozen at the migration cutover point.

This check should be repeated after any future restore-from-backup operation to confirm the restored data isn't older than expected.

### Decommissioning the Old Docker Containers

Not yet performed — the old containers remain stopped on `192.168.5.40` as a safety net. Before deleting them:

1. Confirm at least one full backup cycle of `shared-postgres` (covering the `immich` database) has completed successfully.
2. Confirm `immich-library-pvc` has been included in a storage-level backup/snapshot.
3. Only then remove the stopped containers and their associated volumes on `192.168.5.40`.

## Troubleshooting

### Server Pod CrashLooping

```bash
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl describe pod -n media -l app.kubernetes.io/name=immich-server"
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl logs -n media deploy/immich-server --previous"
```

Common causes: `shared-postgres` unreachable (check `DB_HOSTNAME`/Secret), `immich-library-pvc` not bound, or an incompatible chart/image version pairing.

### Machine Learning Jobs Stuck

```bash
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl logs -n media deploy/immich-machine-learning -f"
```

Check for model download failures (if the `cache` PVC was wiped, first startup after that re-downloads all ONNX models, which can take several minutes).

### Database Connection Errors

```bash
# Confirm shared-postgres is healthy
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl get cluster shared-postgres -n databases"

# Confirm the immich-db-credentials secret matches an existing role
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl get secret immich-db-credentials -n media -o jsonpath='{.data.username}' | base64 -d"
```

### Ingress / TLS Issues

```bash
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl describe ingress immich-server -n media"
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl get certificate -n media | grep immich"
```

### PVC Not Binding

```bash
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl describe pvc immich-library-pvc -n media"
```

Check the NFS-backed storage class/provisioner has capacity for the full `1Ti` request.

## Scaling Considerations

- `immich-server` is not currently horizontally scaled; because job workers run in-process, running multiple replicas requires care around job queue coordination (Valkey-backed BullMQ supports this, but it has not been validated in this deployment).
- `immich-machine-learning` could be scaled independently if ML job throughput becomes a bottleneck (face/CLIP processing on large batch uploads).
- Database scaling is the responsibility of the shared `shared-postgres` cluster, not something tuned per-app.

---

*This deployment guide is part of the Immich documentation for this homelab cluster. It reflects the manifest at `apps/media/immich/helmrelease.yaml` and the live cluster state as of 2026-10-23.*
