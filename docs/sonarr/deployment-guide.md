---
title: Sonarr Deployment Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [deployment, kubernetes, flux, infrastructure, media-automation]
---

# Sonarr Deployment Guide

## Overview

Sonarr is deployed in this cluster as a Flux-managed `HelmRelease` using the `bjw-s` `app-template` chart, in the `media` namespace alongside Radarr, Prowlarr, and FlareSolverr. This guide covers the actual manifest in this repo, how to change it, and how to operate/troubleshoot the live deployment via `kubectl`.

## Prerequisites

### Cluster Requirements

- k3s cluster with Flux CD installed and reconciling this repo
- `bjw-s` Helm repository registered as a Flux `HelmRepository` named `bjw-s` in `flux-system`
- `media` namespace already created (shared with Radarr/Prowlarr/FlareSolverr/Immich)
- A storage class capable of satisfying `ReadWriteOnce` PVCs
- Pre-existing shared PVCs: `downloads-pvc` (completed downloads) and `tv-pvc` (TV library) — these are provisioned separately (see `apps/media/media-pvcs.yaml`), not by the Sonarr `HelmRelease` itself
- Traefik ingress controller and a wildcard TLS secret (`wildcard-conderhome-xyz-tls`) for `*.conderhome.xyz`

### Access Requirements

- `kubectl` access to the cluster, or SSH to the management host:
  ```bash
  ssh hermes-infra@192.168.5.65
  export KUBECONFIG=~/.kube/config
  ```
- Write access to this Git repo for any manifest changes (Flux applies from Git, not from `kubectl apply`)

## Current Deployment

### Manifest Location

```
apps/media/sonarr/
├── helmrelease.yml      # HelmRelease: image, service, ingress, persistence
└── kustomization.yaml   # Kustomize wrapper referencing helmrelease.yml
```

### HelmRelease (as deployed)

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: sonarr
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
      sonarr:
        containers:
          app:
            image:
              repository: linuxserver/sonarr
              tag: 4.0.20 # {"$imagepolicy": "flux-system:sonarr:tag"}
            env:
              TZ: America/Chicago
    service:
      sonarr:
        controller: sonarr
        ports:
          http:
            port: 8989
    ingress:
      sonarr:
        className: traefik
        hosts:
          - host: sonarr.conderhome.xyz
            paths:
              - path: /
                pathType: Prefix
                service:
                  identifier: sonarr
                  port: http
        tls:
          - hosts:
              - sonarr.conderhome.xyz
            secretName: wildcard-conderhome-xyz-tls
    persistence:
      config:
        accessMode: ReadWriteOnce
        size: 1Gi
        globalMounts:
          - path: /config
      media:
        existingClaim: downloads-pvc
        globalMounts:
          - path: /downloads
      tv:
        existingClaim: tv-pvc
        globalMounts:
          - path: /tv/TV Shows
```

### What Flux Creates

From this `HelmRelease`, the `app-template` chart renders:

- `Deployment/sonarr` (namespace `media`) — 1 replica, `strategy: Recreate`
- `Service/sonarr` — ClusterIP, port `8989` named `http`
- `Ingress/sonarr` — Traefik, host `sonarr.conderhome.xyz`, TLS via the wildcard secret
- `PersistentVolumeClaim/sonarr-config` — 1Gi, `ReadWriteOnce`, mounted at `/config`
- No new PVCs for media/tv — these bind to the pre-existing `downloads-pvc` and `tv-pvc` claims via `existingClaim`

## Making Changes

All changes to this app are made by editing `apps/media/sonarr/helmrelease.yml` and letting Flux reconcile — never `kubectl edit` directly, it will be reverted on the next reconciliation.

### Bumping the Image Tag

The `tag:` field carries a Flux image automation marker (`$imagepolicy`), meaning an `ImagePolicy` named `sonarr` in `flux-system` can auto-bump this tag when a new `linuxserver/sonarr` release matches the configured policy. To bump manually instead:

```bash
# Edit apps/media/sonarr/helmrelease.yml, change:
#   tag: 4.0.20 # {"$imagepolicy": "flux-system:sonarr:tag"}
# to the desired tag, keeping the imagepolicy comment intact.

git add apps/media/sonarr/helmrelease.yml
git commit -m "sonarr: bump image to <new-tag>"
git push
```

### Adding Resource Requests/Limits

Sonarr currently has no `resources` block, unlike its sibling Prowlarr (which sets `requests: {cpu: 100m, memory: 128Mi}` / `limits: {cpu: 1000m, memory: 1024Mi}`). Recommended addition:

```yaml
    controllers:
      sonarr:
        containers:
          app:
            image:
              repository: linuxserver/sonarr
              tag: 4.0.20 # {"$imagepolicy": "flux-system:sonarr:tag"}
            env:
              TZ: America/Chicago
            resources:
              requests:
                cpu: 100m
                memory: 256Mi
              limits:
                cpu: 1000m
                memory: 1024Mi
```

### Adding Liveness/Readiness Probes

Not currently set. The `app-template` chart supports probes per-container; `/ping` is unauthenticated and suitable for both:

```yaml
    controllers:
      sonarr:
        containers:
          app:
            probes:
              liveness:
                enabled: true
                custom: true
                spec:
                  httpGet:
                    path: /ping
                    port: 8989
                  initialDelaySeconds: 30
                  periodSeconds: 15
              readiness:
                enabled: true
                custom: true
                spec:
                  httpGet:
                    path: /ping
                    port: 8989
                  initialDelaySeconds: 10
                  periodSeconds: 10
```

### Growing the Config Volume

If `/config` fills up (large SQLite db, verbose logs/backups):

```yaml
    persistence:
      config:
        accessMode: ReadWriteOnce
        size: 2Gi   # increased from 1Gi
        globalMounts:
          - path: /config
```

Note: PVC resize support depends on the underlying StorageClass (`allowVolumeExpansion: true`); check before relying on an in-place resize.

## Deploying / Verifying

### Trigger Reconciliation

Flux reconciles on its own `interval: 1h`, or force it immediately:

```bash
flux reconcile helmrelease sonarr -n media --with-source
```

### Check Rollout Status

```bash
KUBECONFIG=~/.kube/config kubectl get helmrelease sonarr -n media
KUBECONFIG=~/.kube/config kubectl get deploy sonarr -n media
KUBECONFIG=~/.kube/config kubectl rollout status deploy/sonarr -n media
```

### Check Pod Status

```bash
# Find the exact pod label first
KUBECONFIG=~/.kube/config kubectl get pods -n media

# Then describe it
KUBECONFIG=~/.kube/config kubectl describe pod -n media -l app.kubernetes.io/name=sonarr
```

Current live state (reference snapshot):

```
NAME                       READY   STATUS    RESTARTS   AGE
sonarr-7b5d58cbc7-p2hkm    1/1     Running   0          4d

Image:   linuxserver/sonarr:4.0.20
Mounts:  /config (sonarr-config), /downloads (downloads-pvc), /tv/TV Shows (tv-pvc)
```

### Access the App

```bash
curl -sk https://sonarr.conderhome.xyz/ping
```

Expect `{"status":"OK"}`.

## Configuration

### Environment Variables

| Variable | Default (this deployment) | Description |
|----------|---------------------------|-------------|
| `TZ` | `America/Chicago` | Container timezone, affects log timestamps and the calendar/scheduler |
| `PUID` | unset (image default) | LinuxServer.io convention for file-owner UID — not currently overridden |
| `PGID` | unset (image default) | LinuxServer.io convention for file-owner GID — not currently overridden |

### Volume Mounts

| Path | Claim | Purpose |
|------|-------|---------|
| `/config` | `sonarr-config` (dedicated, 1Gi) | Database, settings, logs, API key |
| `/downloads` | `downloads-pvc` (shared with Radarr/download client) | Completed-download staging for import |
| `/tv/TV Shows` | `tv-pvc` (shared library volume) | Final renamed/organized TV library |

### Ingress

| Field | Value |
|-------|-------|
| Class | `traefik` |
| Host | `sonarr.conderhome.xyz` |
| TLS Secret | `wildcard-conderhome-xyz-tls` |
| Backend Service/Port | `sonarr` / `http` (8989) |

## Monitoring and Maintenance

### Health Checks

```bash
# Unauthenticated ping
curl -sk https://sonarr.conderhome.xyz/ping

# Authenticated health (needs API key from /config/config.xml)
curl -sk -H "X-Api-Key: ***" https://sonarr.conderhome.xyz/api/v3/health
```

### Logs

```bash
# Live logs
KUBECONFIG=~/.kube/config kubectl logs -n media deploy/sonarr -f

# Logs persisted on disk (survive pod restarts)
KUBECONFIG=~/.kube/config kubectl exec -n media deploy/sonarr -- ls -la /config/logs
```

### Flux Status

```bash
flux get helmreleases -n media
flux get kustomizations -A | grep media
```

### Backup and Recovery

#### Config/Database Backup

Sonarr's built-in backup (Settings → General → Backups) writes to `/config/Backups`. To pull a copy out of the cluster:

```bash
KUBECONFIG=~/.kube/config kubectl cp media/<sonarr-pod>:/config/Backups ./sonarr-backups-$(date +%Y%m%d)
```

#### Full Config PVC Snapshot

Use whatever cluster-wide PVC/volume snapshot mechanism is already in place for the `media` namespace (e.g., Velero, CSI VolumeSnapshots) to capture `sonarr-config` alongside its siblings.

#### Restore

```bash
KUBECONFIG=~/.kube/config kubectl cp ./sonarr-backups-20261023 media/<sonarr-pod>:/config/Backups
# Then in the web UI: Settings → General → Backups → Restore
```

## Troubleshooting

### Pod Won't Start / CrashLoopBackOff

```bash
KUBECONFIG=~/.kube/config kubectl describe pod -n media -l app.kubernetes.io/name=sonarr
KUBECONFIG=~/.kube/config kubectl logs -n media deploy/sonarr --previous
```

Common causes: corrupted `/config/sonarr.db` after an unclean shutdown, or a PVC mount failure (check PVC is `Bound`).

### HelmRelease Not Reconciling

```bash
flux get helmreleases -n media
KUBECONFIG=~/.kube/config kubectl describe helmrelease sonarr -n media
```

Look for chart version resolution errors against the `bjw-s` `HelmRepository`, or a values schema mismatch after a chart version bump.

### Ingress / TLS Issues

```bash
KUBECONFIG=~/.kube/config kubectl get ingress sonarr -n media -o yaml
KUBECONFIG=~/.kube/config kubectl get certificate -n media   # if cert-manager manages the wildcard
curl -vk https://sonarr.conderhome.xyz/ping
```

### PVC Stuck Pending

```bash
KUBECONFIG=~/.kube/config kubectl get pvc -n media
KUBECONFIG=~/.kube/config kubectl describe pvc sonarr-config -n media
```

Check StorageClass availability and node disk capacity.

### Shared Volume Conflicts

Because `downloads-pvc` and `tv-pvc` are shared with Radarr (and the download client), permission or path-layout changes made by one app's config (e.g., changing folder structure) can affect the others. Coordinate naming/path changes across Sonarr and Radarr when touching shared volumes.

## Scaling Considerations

Sonarr is **not** designed to run multiple replicas — the chart's `strategy: Recreate` and SQLite's single-writer model make horizontal scaling unsafe (you'd get two processes writing the same `sonarr.db` concurrently if `ReadWriteOnce` ever allowed multi-attach, which it doesn't anyway). Scale vertically via `resources.limits` instead if the instance is CPU/memory constrained.

## Removal

To fully remove Sonarr from the cluster:

```bash
git rm -r apps/media/sonarr
# remove the sonarr entry from apps/media/kustomization.yaml if listed there
git commit -m "Remove sonarr"
git push
```

Flux will prune the `HelmRelease`, which deletes the `Deployment`, `Service`, `Ingress`, and the dedicated `sonarr-config` PVC. The shared `downloads-pvc` and `tv-pvc` are untouched since they're `existingClaim` references, not owned by this release.

---

*This deployment guide reflects the live `apps/media/sonarr` manifests in this repo as of 2026-10-23. Cross-reference `architecture.md` for how Sonarr fits into the Prowlarr/FlareSolverr/Radarr stack, and `user-guide.md` / `api-documentation.md` for usage and API detail.*
