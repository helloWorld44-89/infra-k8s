---
title: FlareSolverr Deployment Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [deployment, kubernetes, flux, helm, infrastructure]
---

# FlareSolverr Deployment Guide

## Overview

FlareSolverr is deployed to this k3s cluster via Flux CD as a Helm release using the shared `bjw-s/app-template` chart — the same pattern used across every workload under `apps/media/`. There is no bespoke Dockerfile or standalone manifest in this repo; the deployment is entirely declared in `apps/media/flaresolverr/helmrelease.yml`, reconciled automatically by Flux. This guide covers the actual GitOps deployment as it exists in this cluster, plus the manual steps to reproduce or modify it.

## Prerequisites

- A running k3s cluster with Flux CD installed and the `bjw-s` `HelmRepository` already registered in `flux-system` (shared infra, not specific to this app)
- `kubectl` access to the cluster (via `ssh hermes-infra@192.168.5.65` + `KUBECONFIG=~/.kube/config` in this environment)
- The `media` namespace already exists (shared by all media-stack apps including Prowlarr, which FlareSolverr exists to serve)
- Git access to this repo (`infra-k8s`) for any manifest changes — Flux is the only thing that applies changes to the cluster; direct `kubectl apply`/`edit` against Flux-managed resources will be reverted on the next reconcile

## Current Deployment (GitOps / Flux)

### Manifest Location

```
apps/media/flaresolverr/
├── helmrelease.yml
└── kustomization.yaml
```

### HelmRelease Definition

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

Notable choices in this deployment:
- **No Ingress** — intentionally internal-only; there's nothing for an external client or browser to usefully reach
- **No PersistentVolumeClaim** — fully stateless, no `/config` or data directory mounted
- **No liveness/readiness probes** — not configured in this HelmRelease; health is observed via pod phase only
- **`tag: latest`, no image automation marker** — unlike Prowlarr's `{"$imagepolicy": ...}` comment, FlareSolverr has no Flux image-automation policy wired up; new versions arrive only when the pod is recreated and re-pulls `latest`
- **`CAPTCHA_SOLVER: none`** — deliberately disables third-party CAPTCHA-solving integrations; this instance only handles Cloudflare's JS challenge, not interactive CAPTCHAs

### Deploying / Updating via Flux

Any change to FlareSolverr's config goes through Git, not direct cluster edits:

```bash
# 1. Edit the manifest
vi apps/media/flaresolverr/helmrelease.yml

# 2. Commit and push
git add apps/media/flaresolverr/helmrelease.yml
git commit -m "flaresolverr: adjust config"
git push

# 3. Force an immediate reconcile (optional — otherwise Flux picks it up
#    within the 1h interval, or on the next cluster-wide Flux sync)
ssh hermes-infra@192.168.5.65 \
  "KUBECONFIG=~/.kube/config flux reconcile helmrelease flaresolverr -n media"
```

### Verifying the Rollout

```bash
# HelmRelease status
ssh hermes-infra@192.168.5.65 \
  "KUBECONFIG=~/.kube/config kubectl get helmrelease flaresolverr -n media"

# Pod status
ssh hermes-infra@192.168.5.65 \
  "KUBECONFIG=~/.kube/config kubectl get pods -n media -l app.kubernetes.io/name=flaresolverr"

# Full deployment spec as applied
ssh hermes-infra@192.168.5.65 \
  "KUBECONFIG=~/.kube/config kubectl get deploy flaresolverr -n media -o yaml"
```

Expected healthy state: one pod, `Running`, `1/1 Ready`, image `ghcr.io/flaresolverr/flaresolverr:latest`.

### Service Discovery

The chart creates a ClusterIP Service named `flaresolverr` in the `media` namespace on port `8191`. Internal DNS name for consumers (used by Prowlarr):

```
http://flaresolverr.media.svc.cluster.local:8191
```

Confirm it:

```bash
ssh hermes-infra@192.168.5.65 \
  "KUBECONFIG=~/.kube/config kubectl get svc flaresolverr -n media"
```

## Configuration Reference

### Environment Variables (as set in this deployment)

| Variable | Current value | Description |
|----------|---------------|--------------|
| `LOG_LEVEL` | `info` | Application log verbosity |
| `LOG_FILE` | `none` | Disables writing logs to a file; stdout only |
| `LOG_HTML` | `"false"` | Suppresses dumping full page HTML into logs on failure |
| `CAPTCHA_SOLVER` | `none` | No interactive CAPTCHA-solving backend configured — Cloudflare JS challenges only |
| `TZ` | `Europe/London` | Container timezone |

Other variables FlareSolverr supports upstream (not currently set here, defaults apply): `PORT` (default `8191`), `HEADLESS` (default `true`), `BROWSER_TIMEOUT`.

### Resource Limits

| Type | CPU | Memory |
|------|-----|--------|
| Requests | `100m` | `256Mi` |
| Limits | `1000m` | `1024Mi` |

Headless Chromium is memory-hungry during active challenge-solving; if indexers using FlareSolverr start getting OOMKilled or sluggish, raise the memory limit here first.

### Networking

| Resource | Value |
|----------|-------|
| Service type | ClusterIP (implicit default, no `type` override) |
| Port | `8191` |
| Ingress | None — not exposed outside the cluster |
| Internal DNS | `flaresolverr.media.svc.cluster.local:8191` |

## Manual Reproduction (Outside Flux)

For local testing or a one-off environment, the equivalent raw Kubernetes objects would be:

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flaresolverr
  namespace: media
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: flaresolverr
  template:
    metadata:
      labels:
        app: flaresolverr
    spec:
      containers:
      - name: app
        image: ghcr.io/flaresolverr/flaresolverr:latest
        env:
        - name: LOG_LEVEL
          value: info
        - name: LOG_FILE
          value: none
        - name: LOG_HTML
          value: "false"
        - name: CAPTCHA_SOLVER
          value: none
        - name: TZ
          value: Europe/London
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 1000m
            memory: 1Gi
---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: flaresolverr
  namespace: media
spec:
  selector:
    app: flaresolverr
  ports:
  - port: 8191
    targetPort: 8191
```

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

Equivalent quick local test with plain Docker:

```bash
docker run -d \
  --name flaresolverr \
  -p 8191:8191 \
  -e LOG_LEVEL=info \
  -e CAPTCHA_SOLVER=none \
  -e TZ=Europe/London \
  --restart unless-stopped \
  ghcr.io/flaresolverr/flaresolverr:latest

curl -X POST http://localhost:8191/v1 \
  -H "Content-Type: application/json" \
  -d '{"cmd":"sessions.list"}'
```

## Monitoring and Maintenance

### Health Checks

No HTTP health endpoint is wired into probes in this deployment. Use pod phase and a manual API probe instead:

```bash
# Pod state
ssh hermes-infra@192.168.5.65 \
  "KUBECONFIG=~/.kube/config kubectl get pods -n media -l app.kubernetes.io/name=flaresolverr"

# Functional check (from inside the cluster, e.g. via a debug pod or port-forward)
ssh hermes-infra@192.168.5.65 \
  "KUBECONFIG=~/.kube/config kubectl port-forward -n media svc/flaresolverr 8191:8191" &
curl -s -X POST http://localhost:8191/v1 -H "Content-Type: application/json" \
  -d '{"cmd":"sessions.list"}'
```

### Logs

```bash
ssh hermes-infra@192.168.5.65 \
  "KUBECONFIG=~/.kube/config kubectl logs -n media -l app.kubernetes.io/name=flaresolverr --tail=200 -f"
```

### Restarting

```bash
ssh hermes-infra@192.168.5.65 \
  "KUBECONFIG=~/.kube/config kubectl rollout restart deployment flaresolverr -n media"
```

This also clears any stuck/leaked persistent browser sessions, since sessions are in-memory only.

### Scaling / Resource Tuning

Replicas are not meant to be scaled horizontally — FlareSolverr session state is per-pod and Prowlarr is configured with a single proxy endpoint, so a second replica would just be an unused second browser instance behind round-robin Service routing with its own disjoint session state. If throughput becomes an issue, raise CPU/memory limits in `helmrelease.yml` rather than replica count.

```yaml
resources:
  requests:
    cpu: 200m
    memory: 512Mi
  limits:
    cpu: "2"
    memory: 2Gi
```

Commit the change and let Flux reconcile (or force it as shown above).

## Backup & Recovery

There is nothing to back up — no PVC, no database, no persisted config beyond this Git-tracked HelmRelease. Recovery from any failure state is simply:

```bash
ssh hermes-infra@192.168.5.65 \
  "KUBECONFIG=~/.kube/config kubectl rollout restart deployment flaresolverr -n media"
```

or, if the Deployment itself was deleted/corrupted, Flux will recreate it from `apps/media/flaresolverr/helmrelease.yml` on its next reconcile.

## Troubleshooting

### Pod stuck in `Pending`

```bash
ssh hermes-infra@192.168.5.65 \
  "KUBECONFIG=~/.kube/config kubectl describe pod -n media -l app.kubernetes.io/name=flaresolverr"
```
Check `Events` for scheduling failures (insufficient CPU/memory on nodes, image pull errors).

### Pod `CrashLoopBackOff` / `OOMKilled`

Check `Last State` reason in the describe output above. Chromium under memory pressure is the most common cause — raise the memory limit (see Scaling section).

### HelmRelease not reconciling

```bash
ssh hermes-infra@192.168.5.65 \
  "KUBECONFIG=~/.kube/config kubectl get helmrelease flaresolverr -n media -o yaml"
ssh hermes-infra@192.168.5.65 \
  "KUBECONFIG=~/.kube/config flux logs --kind=HelmRelease --name=flaresolverr -n media"
```

### Prowlarr can't reach the Service

```bash
ssh hermes-infra@192.168.5.65 \
  "KUBECONFIG=~/.kube/config kubectl get svc flaresolverr -n media"
ssh hermes-infra@192.168.5.65 \
  "KUBECONFIG=~/.kube/config kubectl get endpoints flaresolverr -n media"
```
An empty `ENDPOINTS` list means the Service's selector isn't matching the pod's labels (verify the pod is `Ready`) or the pod simply isn't running.

---

*This deployment guide documents the actual Flux-managed FlareSolverr deployment in this cluster (`apps/media/flaresolverr/`), plus equivalent manual steps for reproduction outside GitOps.*
