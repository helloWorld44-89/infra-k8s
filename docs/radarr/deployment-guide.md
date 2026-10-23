---
title: Radarr Deployment Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [deployment, kubernetes, flux, infrastructure]
---

# Radarr Deployment Guide

## Overview

Radarr is deployed in this cluster as a Flux-managed `HelmRelease` using the `bjw-s/app-template` chart, living in the `media` namespace alongside Sonarr, Prowlarr, and FlareSolverr. This guide covers the existing deployment, how to change it, and how to operate/recover it — this is a GitOps-managed app, so changes are made in Git, not with imperative `kubectl` commands against the live cluster.

## Prerequisites

### Cluster Requirements

- **k3s cluster** with Flux CD installed and reconciling this repository
- **Traefik** ingress controller (cluster-wide) with a wildcard TLS secret (`wildcard-conderhome-xyz-tls`) available in the `media` namespace
- **StorageClass** capable of satisfying `ReadWriteOnce` PVC requests
- **Existing PVCs**: `downloads-pvc` and `movies-pvc` must already exist in `media` (shared with Sonarr/download client) — Radarr's manifest references them as `existingClaim`, it does not create them

### Access Requirements

- Git push access to this repository for manifest changes (Flux reconciles from Git — no direct `kubectl apply` for day-to-day changes)
- `kubectl` access to the cluster for operational tasks (logs, exec, describe) via `ssh hermes-infra@192.168.5.65` with `KUBECONFIG=~/.kube/config`

## Current Deployment

### Manifest Location

```
apps/media/radarr/helmrelease.yml
```

### HelmRelease Summary

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: radarr
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
      radarr:
        containers:
          app:
            image:
              repository: linuxserver/radarr
              tag: 6.4.4 # {"$imagepolicy": "flux-system:radarr:tag"}
            env:
              TZ: America/Chicago
    service:
      radarr:
        controller: radarr
        ports:
          http:
            port: 7878
    ingress:
      radarr:
        className: traefik
        hosts:
          - host: radarr.conderhome.xyz
            paths:
              - path: /
                pathType: Prefix
                service:
                  identifier: radarr
                  port: http
        tls:
          - hosts:
              - radarr.conderhome.xyz
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
      movies:
        existingClaim: movies-pvc
        globalMounts:
          - path: /movies
```

### Live State (reference)

As of this writing:

- **Deployment**: `radarr` in namespace `media`, 1/1 ready, strategy `Recreate`
- **Pod**: `radarr-5c854b7599-zj7sb`, image `linuxserver/radarr:6.4.4`, running on node `prod-k3s-2`
- **Volumes**: `radarr-config` (PVC, config), `downloads-pvc` (shared), `movies-pvc` (dedicated)
- **No resource requests/limits set** and **no liveness/readiness probes defined** — both are candidate improvements (see Hardening below)

## Making Changes

All changes to this deployment go through Git — edit `apps/media/radarr/helmrelease.yml`, commit, and push; Flux reconciles within the hour, or immediately with a manual trigger (see below).

### Common Changes

#### Bump the Image Tag Manually

```yaml
image:
  repository: linuxserver/radarr
  tag: 6.5.0 # {"$imagepolicy": "flux-system:radarr:tag"}
```

Note the `$imagepolicy` marker — if Flux image automation is active for `radarr`, it will overwrite manual tag edits on its own schedule. Check `flux-system` for an `ImagePolicy`/`ImageRepository` named `radarr` before hand-editing tags.

#### Add Resource Requests/Limits

```yaml
controllers:
  radarr:
    containers:
      app:
        image:
          repository: linuxserver/radarr
          tag: 6.4.4
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

(Matches the pattern already used by the sibling Prowlarr and FlareSolverr manifests in this repo.)

#### Add Liveness/Readiness Probes

```yaml
controllers:
  radarr:
    containers:
      app:
        probes:
          liveness:
            enabled: true
            custom: true
            spec:
              httpGet:
                path: /ping
                port: 7878
              initialDelaySeconds: 30
              periodSeconds: 20
          readiness:
            enabled: true
            custom: true
            spec:
              httpGet:
                path: /ping
                port: 7878
              initialDelaySeconds: 10
              periodSeconds: 10
```

Radarr exposes an unauthenticated `/ping` endpoint suitable for basic liveness/readiness checks.

#### Increase Config Volume Size

```yaml
persistence:
  config:
    accessMode: ReadWriteOnce
    size: 2Gi
    globalMounts:
      - path: /config
```

Note: growing a PVC generally requires the underlying StorageClass to support volume expansion; shrinking is not supported by Kubernetes and requires a new PVC + data migration.

### Applying Changes

```bash
git add apps/media/radarr/helmrelease.yml
git commit -m "radarr: <describe change>"
git push
```

Flux reconciles automatically within `spec.interval` (1h). To force an immediate reconcile:

```bash
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config flux reconcile helmrelease radarr -n media --with-source"
```

## Operations

### Checking Status

```bash
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl get pods -n media -l app.kubernetes.io/name=radarr"
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl get helmrelease radarr -n media"
```

### Viewing Logs

```bash
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl logs -n media -l app.kubernetes.io/name=radarr --tail=200"

# follow
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl logs -n media -l app.kubernetes.io/name=radarr -f"
```

### Restarting

Radarr uses the `Recreate` strategy, so a manual rollout restart fully tears down and recreates the single pod (brief downtime, avoids SQLite contention):

```bash
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl rollout restart deployment/radarr -n media"
```

### Shell / Exec Access

```bash
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl exec -it -n media deploy/radarr -- /bin/sh"
```

### Retrieving the API Key

```bash
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl exec -n media deploy/radarr -- grep ApiKey /config/config.xml"
```

Treat the output as a secret — do not paste it into chat, tickets, or logs.

## Backup and Recovery

### Config Backup (PVC-level)

The entire Radarr state (database, settings, API key) lives on the `radarr-config` PVC. Snapshot or back up this PVC using your cluster's volume snapshot/backup tooling (e.g. Velero, Longhorn snapshots) to enable full-instance recovery.

### In-App Backup

Radarr can self-backup its database/config to `/config/Backups` on a schedule (Settings → General → Backups). Combine with an external volume backup to get these snapshots off-cluster too.

### Movie Library Backup

`/movies` (backed by `movies-pvc`) holds the actual media files and is typically large — back it up with your normal bulk-storage backup process rather than frequent snapshots.

### Recovery

1. Ensure `downloads-pvc` and `movies-pvc` still exist and are correctly referenced
2. Restore `radarr-config` from the most recent PVC snapshot/backup (or restore a `/config/Backups` archive onto a fresh PVC)
3. Re-apply the HelmRelease (Flux will do this automatically once Git state matches) and verify the pod comes up `1/1 Ready`
4. Confirm in the UI that movies, indexers (re-synced from Prowlarr if needed), and queue history look correct

## Monitoring and Maintenance

### Health Checks

```bash
# From inside the cluster
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl exec -n media deploy/radarr -- curl -s http://localhost:7878/ping"

# Via the API (requires API key)
curl -s -H "X-Api-Key: ***" https://radarr.conderhome.xyz/api/v3/health
```

### Pod-Level Diagnostics

```bash
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl describe pod -n media -l app.kubernetes.io/name=radarr"
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl top pod -n media -l app.kubernetes.io/name=radarr"
```

## Troubleshooting

### Pod Won't Start

```bash
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl describe pod -n media -l app.kubernetes.io/name=radarr"
```

Check for PVC binding failures (`downloads-pvc`/`movies-pvc` missing or in a different access mode) and image pull errors.

### HelmRelease Not Reconciling

```bash
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config flux get helmrelease radarr -n media"
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl describe helmrelease radarr -n media"
```

Look for chart source errors (HelmRepository `bjw-s` unreachable) or values validation failures.

### Ingress / TLS Issues

```bash
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl get ingress -n media radarr -o yaml"
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl get secret -n media wildcard-conderhome-xyz-tls"
```

Confirm the wildcard TLS secret is present in `media` and not expired.

### Database Corruption

```bash
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl exec -n media deploy/radarr -- ls -la /config"
```

If `radarr.db` is corrupted, stop the pod, restore from a config backup/snapshot, or (last resort) delete the db file and let Radarr rebuild a fresh database — this loses history, custom formats, and quality profile customizations, so prefer a backup restore.

## Hardening Recommendations (not yet applied)

These are gaps identified against the sibling manifests (Prowlarr, FlareSolverr) in this repo, worth considering for a future change:

1. **Add resource requests/limits** — currently unset, unlike Prowlarr/FlareSolverr
2. **Add liveness/readiness probes** against `/ping` — currently none defined
3. **Consider pinning `app-template` to an exact chart version** rather than the `3.x` floating range for more predictable upgrades

---

*This deployment guide covers the Radarr HelmRelease in `apps/media/radarr/helmrelease.yml`. For usage and API details, see the user guide and API documentation alongside this guide.*
