---
title: Prowlarr Deployment Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [deployment, kubernetes, flux, infrastructure, media]
---

# Prowlarr Deployment Guide

## Overview

Prowlarr is deployed into this homelab's k3s cluster via Flux CD using the `bjw-s` `app-template` Helm chart, as a `HelmRelease` resource in the `media` namespace. It runs alongside — and depends on — FlareSolverr for Cloudflare/CAPTCHA-protected indexers. This guide covers the actual manifests used in this repository, plus general operational and troubleshooting commands.

## Prerequisites

### Cluster Requirements

- k3s cluster with Flux CD v2 installed and reconciling this repository
- `media` namespace already present (shared with Radarr, Sonarr, FlareSolverr, etc.)
- `bjw-s` `HelmRepository` registered in `flux-system` (chart `app-template`, version `3.x`)
- A default `StorageClass` capable of satisfying `ReadWriteOnce` PVCs
- Traefik ingress controller and cert-manager with a wildcard cert for `*.conderhome.xyz` (secret `wildcard-conderhome-xyz-tls`)

### Dependencies

- **FlareSolverr** (`apps/media/flaresolverr/helmrelease.yml`) should be deployed in the same namespace before Prowlarr indexers requiring it are configured — Prowlarr itself will deploy fine without it, but any indexer relying on the FlareSolverr proxy will fail tests until FlareSolverr is up.
- **Radarr / Sonarr** should already be deployed if you intend to use Prowlarr's application-sync feature immediately after deploy.

## Repository Layout

```
apps/media/
├── prowlarr/
│   ├── helmrelease.yml
│   └── kustomization.yaml
├── flaresolverr/
│   ├── helmrelease.yml
│   └── kustomization.yaml
├── radarr/
└── sonarr/
```

## Current Manifest

`apps/media/prowlarr/helmrelease.yml`:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: prowlarr
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
      prowlarr:
        containers:
          app:
            image:
              repository: linuxserver/prowlarr
              tag: 2.6.5 # {"$imagepolicy": "flux-system:prowlarr:tag"}
            env:
              TZ: America/Chicago
            resources:
              requests:
                cpu: 100m
                memory: 128Mi
              limits:
                cpu: 1000m
                memory: 1024Mi
    service:
      prowlarr:
        controller: prowlarr
        ports:
          http:
            port: 9696
    ingress:
      prowlarr:
        className: traefik
        hosts:
          - host: prowlarr.conderhome.xyz
            paths:
              - path: /
                pathType: Prefix
                service:
                  identifier: prowlarr
                  port: http
        tls:
          - hosts:
              - prowlarr.conderhome.xyz
            secretName: wildcard-conderhome-xyz-tls
    persistence:
      config:
        accessMode: ReadWriteOnce
        size: 1Gi
        globalMounts:
          - path: /config
```

Companion `apps/media/flaresolverr/helmrelease.yml` (relevant excerpt):

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: flaresolverr
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
      flaresolverr:
        containers:
          app:
            image:
              repository: ghcr.io/flaresolverr/flaresolverr
              tag: latest
            env:
              LOG_LEVEL: info
              LOG_FILE: none
              LOG_HTML: "false"
              CAPTCHA_SOLVER: none
              TZ: Europe/London
            resources:
              requests:
                cpu: 100m
                memory: 256Mi
              limits:
                cpu: 1000m
                memory: 1024Mi
    service:
      flaresolverr:
        controller: flaresolverr
        ports:
          http:
            port: 8191
```

Note FlareSolverr intentionally has **no `ingress` block** — it is cluster-internal only, consumed by Prowlarr at `http://flaresolverr.media.svc.cluster.local:8191` (or short form `http://flaresolverr:8191` from within the `media` namespace).

## Deployment via Flux (GitOps)

This is the only supported deployment path in this repository — manifests are reconciled automatically by Flux, not applied manually.

### 1. Make Changes

Edit `apps/media/prowlarr/helmrelease.yml` (or `flaresolverr/helmrelease.yml`) in a feature branch.

### 2. Validate Locally

```bash
# Lint/validate YAML
yamllint apps/media/prowlarr/helmrelease.yml

# Dry-run template rendering (requires flux CLI + cluster context)
flux build helmrelease prowlarr \
  --path ./apps/media/prowlarr \
  --namespace media
```

### 3. Commit and Push

```bash
git add apps/media/prowlarr/helmrelease.yml
git commit -m "prowlarr: <description of change>"
git push
```

### 4. Reconcile

Flux reconciles on its normal interval (`spec.interval: 1h`), or force it immediately:

```bash
flux reconcile helmrelease prowlarr -n media --with-source
```

### 5. Verify Rollout

```bash
ssh hermes-infra@192.168.5.65 \
  "KUBECONFIG=~/.kube/config kubectl get pods -n media -l app.kubernetes.io/name=prowlarr"

ssh hermes-infra@192.168.5.65 \
  "KUBECONFIG=~/.kube/config kubectl rollout status deploy/prowlarr -n media"
```

## Configuration Reference

### Environment Variables

| Variable | Current Value | Description |
|----------|---------------|--------------|
| `TZ` | `America/Chicago` | Container timezone (affects scheduled task/log timestamps) |

### Image Updates

The image tag is managed by Flux's image-automation controller via the marker comment:

```yaml
tag: 2.6.5 # {"$imagepolicy": "flux-system:prowlarr:tag"}
```

Flux's `ImagePolicy`/`ImageRepository` resources (defined elsewhere in the cluster's `flux-system` configuration) detect new `linuxserver/prowlarr` tags and open an automated commit bumping this value — manual tag bumps in this file will be overwritten on the next automation pass unless the policy is paused.

### Resource Limits

| Resource | Request | Limit |
|----------|---------|-------|
| CPU | 100m | 1000m |
| Memory | 128Mi | 1024Mi |

### Persistence

| PVC | Mount Path | Access Mode | Size |
|-----|-----------|-------------|------|
| `prowlarr-config` | `/config` | ReadWriteOnce | 1Gi |

Because persistence is `ReadWriteOnce` and the deployment strategy is `Recreate`, Prowlarr cannot run with more than one replica, and a rollout briefly terminates the old pod before starting the new one (short downtime window, acceptable for this workload).

### Networking

| Resource | Value |
|----------|-------|
| Service | `prowlarr` (ClusterIP, port 9696) |
| In-cluster FQDN | `prowlarr.media.svc.cluster.local:9696` |
| Ingress host | `prowlarr.conderhome.xyz` |
| Ingress class | `traefik` |
| TLS secret | `wildcard-conderhome-xyz-tls` |
| FlareSolverr dependency | `flaresolverr.media.svc.cluster.local:8191` (ClusterIP, no Ingress) |

## Monitoring and Maintenance

### Health Checks

```bash
# Pod status
ssh hermes-infra@192.168.5.65 \
  "KUBECONFIG=~/.kube/config kubectl get pods -n media -l app.kubernetes.io/name=prowlarr"

# Describe pod (events, restarts, probe failures)
ssh hermes-infra@192.168.5.65 \
  "KUBECONFIG=~/.kube/config kubectl describe pod -n media -l app.kubernetes.io/name=prowlarr"

# App-level health via its own API (see api-documentation.md)
curl -s -H "X-Api-Key: ***" https://prowlarr.conderhome.xyz/api/v1/health
```

### Log Management

```bash
# Prowlarr logs
ssh hermes-infra@192.168.5.65 \
  "KUBECONFIG=~/.kube/config kubectl logs -n media -l app.kubernetes.io/name=prowlarr --tail=200"

# FlareSolverr logs (check when indexer-proxy tests fail)
ssh hermes-infra@192.168.5.65 \
  "KUBECONFIG=~/.kube/config kubectl logs -n media -l app.kubernetes.io/name=flaresolverr --tail=200"
```

### Backup and Recovery

Prowlarr's entire state (indexer definitions, credentials, app connections, FlareSolverr proxy config) lives in the `prowlarr-config` PVC. There is no external database — back up the PVC itself.

```bash
# Example: snapshot the config PVC contents via a one-off debug pod
ssh hermes-infra@192.168.5.65 \
  "KUBECONFIG=~/.kube/config kubectl exec -n media deploy/prowlarr -- tar czf - -C /config . " \
  > prowlarr-config-backup-$(date +%Y%m%d).tar.gz
```

Restoring is the reverse: scale the deployment to 0, extract the backup archive back into a fresh PVC mount, scale back up.

## Troubleshooting

### Pod Won't Start

```bash
ssh hermes-infra@192.168.5.65 \
  "KUBECONFIG=~/.kube/config kubectl describe pod -n media -l app.kubernetes.io/name=prowlarr"
ssh hermes-infra@192.168.5.65 \
  "KUBECONFIG=~/.kube/config kubectl get events -n media --sort-by=.lastTimestamp | tail -30"
```

Common causes: PVC stuck `Pending` (StorageClass/provisioner issue), image pull failure, or a prior pod stuck terminating (Recreate strategy waits for full termination before starting the new pod).

### FlareSolverr Connectivity Failures

```bash
# Confirm FlareSolverr is up
ssh hermes-infra@192.168.5.65 \
  "KUBECONFIG=~/.kube/config kubectl get pods -n media -l app.kubernetes.io/name=flaresolverr"

# Test FlareSolverr directly from inside the cluster
ssh hermes-infra@192.168.5.65 \
  "KUBECONFIG=~/.kube/config kubectl run flaresolverr-test --rm -it --restart=Never -n media \
   --image=curlimages/curl -- curl -s http://flaresolverr:8191"
```

If this curl fails, the problem is FlareSolverr itself (check its pod logs); if it succeeds but Prowlarr's indexer-proxy test still fails, re-check the `Host` field configured in Prowlarr's **Settings → Indexers → Indexer Proxies** for typos or a stale IP instead of the service DNS name.

### Ingress / TLS Issues

```bash
ssh hermes-infra@192.168.5.65 \
  "KUBECONFIG=~/.kube/config kubectl get ingress -n media prowlarr -o yaml"
curl -vI https://prowlarr.conderhome.xyz
```

## Scaling Considerations

Prowlarr is not designed to be horizontally scaled in this topology — a single SQLite-backed instance with `ReadWriteOnce` storage is the expected and sufficient deployment shape for a homelab. If load or availability ever become a concern, the correct path is migrating Prowlarr's backing store rather than adding replicas, which is out of scope for the current manifest.

---

*This deployment guide covers the Prowlarr HelmRelease as committed in this repository (`apps/media/prowlarr/helmrelease.yml`) and its FlareSolverr dependency (`apps/media/flaresolverr/helmrelease.yml`).*
