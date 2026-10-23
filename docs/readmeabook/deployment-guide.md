---
title: ReadMeABook Deployment Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [deployment, kubernetes, flux, infrastructure, audiobook]
---

# ReadMeABook Deployment Guide

## Overview

ReadMeABook runs in this cluster as a Flux-managed `HelmRelease` using the generic `bjw-s` `app-template` chart, in the `media` namespace. The manifest lives at `apps/media/readmeabook/helmrelease.yml`. This guide documents the actual deployment as configured and observed in this k3s cluster — it is not a generic "how to deploy ReadMeABook anywhere" guide.

## Prerequisites (as implemented here)

- k3s cluster with Flux CD (`helm.toolkit.fluxcd.io/v2`) and the `bjw-s` HelmRepository already registered in `flux-system`
- `local-path` StorageClass available (used for all app-private PVCs)
- Pre-existing NFS-backed PVCs `downloads-pvc` and `audiobooks-pvc` already provisioned in the `media` namespace (shared with sibling media apps — not created by this HelmRelease)
- Traefik ingress controller with the cluster's shared wildcard TLS secret `wildcard-conderhome-xyz-tls` already present in `media`

## Manifest

Source: `apps/media/readmeabook/helmrelease.yml`

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: readmeabook
  namespace: media
spec:
  interval: 1h
  chart:
    spec:
      chart: app-template
      version: "3.x"
      sourceRef:
        kind: HelmRepository
        name: bjw-s
        namespace: flux-system
  values:
    controllers:
      readmeabook:
        containers:
          app:
            image:
              repository: ghcr.io/kikootwo/readmeabook
              tag: latest
            env:
              PUID: "1000"
              PGID: "1000"
              PUBLIC_URL: "https://audiobook.conderhome.xyz"
            probes:
              liveness:
                enabled: true
                custom: true
                spec:
                  httpGet:
                    path: /api/health
                    port: 3030
                  initialDelaySeconds: 60
                  periodSeconds: 30
                  timeoutSeconds: 10
                  failureThreshold: 3
              readiness:
                enabled: true
                custom: true
                spec:
                  httpGet:
                    path: /api/health
                    port: 3030
                  initialDelaySeconds: 5
                  periodSeconds: 10
            resources:
              requests:
                cpu: 100m
                memory: 512Mi
              limits:
                cpu: 1000m
                memory: 2Gi
    service:
      readmeabook:
        controller: readmeabook
        ports:
          http:
            port: 3030
    ingress:
      readmeabook:
        className: traefik
        hosts:
          - host: audiobook.conderhome.xyz
            paths:
              - path: /
                pathType: Prefix
                service:
                  identifier: readmeabook
                  port: http
        tls:
          - hosts:
              - audiobook.conderhome.xyz
            secretName: wildcard-conderhome-xyz-tls
    persistence:
      config:
        accessMode: ReadWriteOnce
        storageClass: local-path
        size: 1Gi
        globalMounts:
          - path: /app/config
      cache:
        accessMode: ReadWriteOnce
        storageClass: local-path
        size: 5Gi
        globalMounts:
          - path: /app/cache
      pgdata:
        accessMode: ReadWriteOnce
        storageClass: local-path
        size: 5Gi
        globalMounts:
          - path: /var/lib/postgresql/data
      redis:
        accessMode: ReadWriteOnce
        storageClass: local-path
        size: 1Gi
        globalMounts:
          - path: /var/lib/redis
      downloads:
        existingClaim: downloads-pvc
        globalMounts:
          - path: /downloads
      media:
        existingClaim: audiobooks-pvc
        globalMounts:
          - path: /media
```

## Configuration Reference

### Environment Variables (as set)

| Variable | Value | Purpose |
|----------|-------|---------|
| `PUID` | `1000` | Container process UID, for file ownership on NFS/local-path mounts |
| `PGID` | `1000` | Container process GID |
| `PUBLIC_URL` | `https://audiobook.conderhome.xyz` | External URL the app uses to build links/callbacks |

### Volume Mounts

| Mount | Backing | Size | StorageClass | Shared? |
|-------|---------|------|---------------|---------|
| `/app/config` | `readmeabook-config` PVC | 1Gi | local-path | No — app-private |
| `/app/cache` | `readmeabook-cache` PVC | 5Gi | local-path | No — app-private |
| `/var/lib/postgresql/data` | `readmeabook-pgdata` PVC | 5Gi | local-path | No — dedicated Postgres, NOT the shared CNPG cluster |
| `/var/lib/redis` | `readmeabook-redis` PVC | 1Gi | local-path | No — dedicated Redis |
| `/downloads` | `downloads-pvc` (existing claim) | 500Gi | NFS, RWX | Yes — shared with other media apps |
| `/media` | `audiobooks-pvc` (existing claim) | 500Gi | NFS, RWX | Yes — shared library volume |

**Important**: because `pgdata` and `redis` are `local-path` (node-local, ReadWriteOnce), the pod is pinned to whichever node currently hosts those volumes. At time of writing that is `prod-k3s-3`. Losing that node's local disk without a separate backup loses the app's Postgres database and Redis state — the two shared NFS volumes (`downloads`, `media`) are unaffected since they live on the NFS server, not the node.

### Networking

- **Service**: ClusterIP, name `readmeabook`, port `http` = 3030
- **Ingress**: Traefik `IngressRoute`-style config via app-template's generic ingress, host `audiobook.conderhome.xyz`, path `/` prefix
- **TLS**: reuses cluster-wide `wildcard-conderhome-xyz-tls` secret — no dedicated `Certificate` resource for this app

### Probes

| Probe | Path | Port | Initial Delay | Period | Timeout | Failure Threshold |
|-------|------|------|----------------|--------|---------|---------------------|
| Liveness | `/api/health` | 3030 | 60s | 30s | 10s | 3 |
| Readiness | `/api/health` | 3030 | 5s | 10s | 1s | 3 |

### Resources

| | Requests | Limits |
|---|----------|--------|
| CPU | 100m | 1000m |
| Memory | 512Mi | 2Gi |

Live observed usage: ~3m CPU / ~374Mi memory at idle (single pod, `readmeabook-65fcff6985-mjf4z`, node `prod-k3s-3`, 1/1 Running).

## Deploying / Updating

This app is entirely GitOps-managed. There is no manual `kubectl apply` path for normal changes:

1. Edit `apps/media/readmeabook/helmrelease.yml` in this repo
2. Commit and push
3. Flux reconciles within the `interval: 1h` window, or force it immediately:
   ```bash
   flux reconcile helmrelease readmeabook -n media
   ```

### Image Updates

`image.tag: latest` with `imagePullPolicy: Always` means new upstream images are picked up automatically on pod recreation (e.g. during a Flux reconcile that touches the pod spec, or a manual rollout restart) — there is no pinned version and no Flux image-automation policy wired up for this app. To force picking up a newer `latest` build without waiting for a natural pod recreation:

```bash
kubectl rollout restart deployment/readmeabook -n media
```

Because `strategy.type` is `Recreate` (required by the single-writer `local-path` PVCs), expect a brief outage during any restart — the old pod is fully terminated before the new one starts, there is no overlap.

## Operational Commands

```bash
# Pod status
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl get pods -n media -l app.kubernetes.io/name=readmeabook -o wide"

# Full deployment spec (live, as reconciled)
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl get deploy readmeabook -n media -o yaml"

# Describe pod (events, volumes, probe status)
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl describe pod -n media -l app.kubernetes.io/name=readmeabook"

# Resource usage
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl top pod -n media -l app.kubernetes.io/name=readmeabook"

# Logs
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl logs -n media deploy/readmeabook -f"

# PVC status
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl get pvc -n media | grep readmeabook"

# Health check via ingress
curl -s https://audiobook.conderhome.xyz/api/health
```

## Backup & Recovery

ReadMeABook's Postgres database is **not** covered by the cluster's shared CNPG backup routine — it is a self-managed Postgres instance on a `local-path` PVC. Treat it as needing its own backup plan:

```bash
# Dump the embedded Postgres from inside the pod
kubectl exec -n media deploy/readmeabook -- pg_dump -U <configured-user> <configured-db> > readmeabook-backup-$(date +%Y%m%d).sql

# Restore (into a fresh/matching instance)
kubectl exec -i -n media deploy/readmeabook -- psql -U <configured-user> <configured-db> < readmeabook-backup-YYYYMMDD.sql
```

This deployment's data was itself originally migrated this way: the prior standalone Docker deployment on host `192.168.5.40` was dumped with `pg_dump` and restored into this pod's Postgres with `pg_restore`, confirmed by row-count parity (44 rows in the `requests` table) after migration. The same `pg_dump`/`pg_restore` approach is the supported path for ongoing backups or any future re-migration.

The `config` and `cache` PVCs are not database-backed and can be backed up as plain files via any node-level PVC snapshot/backup tooling already used elsewhere in the cluster for `local-path` volumes, if one exists; otherwise they are reproducible by re-running the setup wizard (minus request history, which lives only in Postgres).

## Known Trade-offs / Things to Watch

- **No HA**: single replica, `Recreate` strategy — any pod move or restart is a brief full outage.
- **Node pinning**: `local-path` PVCs (`config`, `cache`, `pgdata`, `redis`) tie the pod to `prod-k3s-3`. Draining or losing that node needs a plan for these four volumes.
- **`latest` tag**: no version pinning — an upstream breaking change ships straight into this deployment on the next pod recreation. If stability becomes an issue, pin `image.tag` to a specific release.
- **Isolated Postgres/Redis**: outside the cluster's shared CNPG backup/HA story by design; needs its own backup discipline (see above).

---

*This guide reflects the exact Flux HelmRelease deployed in this cluster (`apps/media/readmeabook/helmrelease.yml`) and live state captured via SSH/kubectl against `192.168.5.65`. For upstream application configuration (Prowlarr/qBittorrent/SABnzbd/Plex settings), see the in-app Settings pages and the project repository (kikootwo/ReadMeABook on GitHub).*
