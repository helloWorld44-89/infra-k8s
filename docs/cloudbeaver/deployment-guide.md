---
title: CloudBeaver Deployment Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [deployment, kubernetes, flux, infrastructure]
---

# CloudBeaver Deployment Guide

## Overview

CloudBeaver is deployed in this homelab's k3s cluster via Flux GitOps, using the `bjw-s/app-template` Helm chart. This guide covers the current live deployment, how it's managed through the repo, and how to operate/modify it.

## Prerequisites

### System Requirements

- **CPU/Memory**: No explicit requests/limits are set in the current manifest (`resources: {}`); runs comfortably on modest node headroom
- **Storage**: 1Gi PersistentVolumeClaim (NFS storage class) for the workspace
- **Network**: Cluster-internal access to target database Services (shared-postgres, grafana-postgres, mysql, app DBs); Traefik Ingress for external access

### Cluster Dependencies

- **Flux**: GitOps controller reconciling the HelmRelease
- **HelmRepository `bjw-s`**: Must exist in `flux-system` namespace, providing the `app-template` chart
- **Traefik**: Ingress controller (`className: traefik`)
- **NFS storage class**: Backing the `cloudbeaver-data` PVC
- **`cloudbeaver-secrets` Secret**: Must exist in the `tools` namespace before the pod can start successfully (provides `CB_ADMIN_NAME` / `CB_ADMIN_PASSWORD`)

## Current Live Deployment

This is managed entirely through the repo at `apps/tools/cloudbeaver/helmrelease.yml` — changes are made via git, not `kubectl edit`.

### HelmRelease Summary

| Field | Value |
|-------|-------|
| Namespace | `tools` |
| Chart | `app-template` (bjw-s), version `3.x` |
| Image | `dbeaver/cloudbeaver:25.0.1` |
| Service port | `8978` (ClusterIP) |
| Ingress host | `cloudbeaver.conderhome.xyz` (Traefik) |
| Persistence | `cloudbeaver-data` PVC, 1Gi, ReadWriteOnce, mounted at `/opt/cloudbeaver/workspace` |
| Replicas | 1 (Deployment strategy: `Recreate`) |

### Manifest

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: cloudbeaver
  namespace: tools
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
      cloudbeaver:
        containers:
          app:
            image:
              repository: dbeaver/cloudbeaver
              tag: "25.0.1"
            env:
              CB_ADMIN_NAME:
                valueFrom:
                  secretKeyRef:
                    name: cloudbeaver-secrets
                    key: CB_ADMIN_NAME
              CB_ADMIN_PASSWORD:
                valueFrom:
                  secretKeyRef:
                    name: cloudbeaver-secrets
                    key: CB_ADMIN_PASSWORD
    service:
      app:
        controller: cloudbeaver
        ports:
          http:
            port: 8978
    persistence:
      data:
        accessMode: ReadWriteOnce
        size: 1Gi
        globalMounts:
          - path: /opt/cloudbeaver/workspace
    ingress:
      app:
        className: traefik
        hosts:
          - host: cloudbeaver.conderhome.xyz
            paths:
              - path: /
                pathType: Prefix
                service:
                  identifier: app
                  port: http
```

### Required Secret

The `cloudbeaver-secrets` Secret must exist in `tools` before CloudBeaver will start cleanly:

```bash
kubectl create secret generic cloudbeaver-secrets \
  --namespace tools \
  --from-literal=CB_ADMIN_NAME=admin \
  --from-literal=CB_ADMIN_PASSWORD='<strong-password>' \
  --dry-run=client -o yaml | kubectl apply -f -
```

In a GitOps-managed cluster, prefer sealing this with whatever secret-encryption mechanism the repo already uses (e.g. SOPS) rather than applying it ad hoc, so it is reproducible from git.

## Making Changes

### Standard GitOps Flow

1. Edit `apps/tools/cloudbeaver/helmrelease.yml` in this repo
2. Commit and push
3. Flux reconciles within the `interval: 1h` window, or force it immediately:
   ```bash
   flux reconcile helmrelease cloudbeaver -n tools
   ```
4. Verify rollout:
   ```bash
   kubectl get pods -n tools -l app.kubernetes.io/name=cloudbeaver -w
   ```

### Common Changes

**Upgrading the image tag**
```yaml
image:
  repository: dbeaver/cloudbeaver
  tag: "<new-version>"   # check dbeaver/cloudbeaver on Docker Hub for available tags
```
Because the Deployment strategy is `Recreate`, the existing pod is terminated before the new one starts — expect a short outage during upgrades.

**Growing the workspace volume**
```yaml
persistence:
  data:
    accessMode: ReadWriteOnce
    size: 2Gi   # increase as needed
```
Note: PVC resizing support depends on the NFS storage class's capabilities; test before relying on in-place resize for production-adjacent clusters.

**Adding resource limits** (currently unset)
```yaml
containers:
  app:
    resources:
      requests:
        cpu: 100m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 512Mi
```

## Verifying the Deployment

```bash
# Pod status
kubectl get pods -n tools -l app.kubernetes.io/name=cloudbeaver -o wide

# Pod details (events, image, mounts)
kubectl describe pod -n tools -l app.kubernetes.io/name=cloudbeaver

# Deployment rollout state
kubectl get deploy cloudbeaver -n tools -o yaml

# Service and ingress
kubectl get svc -n tools | grep cloudbeaver
kubectl get ingress -n tools | grep cloudbeaver

# PVC status
kubectl get pvc -n tools | grep cloudbeaver

# Logs
kubectl logs -n tools -l app.kubernetes.io/name=cloudbeaver -f
```

Current observed state (at time of writing): 1/1 Running, PVC `cloudbeaver-data` Bound on the `nfs` storage class, ClusterIP Service on port 8978, Deployment condition `Available=True`.

## Monitoring and Maintenance

### Health Checks

No liveness/readiness probes are defined in the current manifest — health is inferred from Deployment status and pod `Running`/`Ready` state rather than an HTTP health check. A lightweight manual check:

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://cloudbeaver.conderhome.xyz
```

Consider adding probes if this becomes a concern:
```yaml
containers:
  app:
    probes:
      liveness:
        enabled: true
        custom: true
        spec:
          httpGet:
            path: /
            port: 8978
          initialDelaySeconds: 30
          periodSeconds: 10
      readiness:
        enabled: true
        custom: true
        spec:
          httpGet:
            path: /
            port: 8978
          initialDelaySeconds: 10
          periodSeconds: 10
```
(Exact probe syntax depends on the `app-template` chart version's schema — verify against the chart's values reference before applying.)

### Log Management

```bash
kubectl logs -n tools -l app.kubernetes.io/name=cloudbeaver
kubectl logs -n tools -l app.kubernetes.io/name=cloudbeaver -f
kubectl logs -n tools -l app.kubernetes.io/name=cloudbeaver > cloudbeaver-logs.txt
```

### Backup and Recovery

#### Workspace Backup

The `cloudbeaver-data` PVC holds connection definitions, user accounts, and settings — back it up like any other NFS-backed PVC in the cluster's standard volume backup routine. There's no CloudBeaver-specific export/import tooling beyond manually re-creating connections if the volume is lost.

#### Restore

1. Ensure the PVC is restored/re-provisioned with prior contents (or accept a clean workspace)
2. Re-apply the HelmRelease (Flux will reconcile automatically if the manifest hasn't changed)
3. Re-create the `cloudbeaver-secrets` Secret if it was lost
4. Log in and re-register any database connections if the workspace was not restored from backup

## Troubleshooting

### Pod Won't Start

```bash
kubectl describe pod -n tools -l app.kubernetes.io/name=cloudbeaver
```
Common causes:
- `cloudbeaver-secrets` Secret missing or missing a key (`CB_ADMIN_NAME`/`CB_ADMIN_PASSWORD`) → `CreateContainerConfigError`
- PVC stuck `Pending` → check NFS storage class/provisioner health
- Image pull issues → check `imagePullPolicy: Always` against registry reachability

### HelmRelease Not Reconciling

```bash
flux get helmrelease cloudbeaver -n tools
flux logs --kind=HelmRelease --name=cloudbeaver -n tools
```

### Ingress Not Routing

```bash
kubectl get ingress -n tools
kubectl describe ingress -n tools -l app.kubernetes.io/name=cloudbeaver
```
Confirm Traefik is healthy cluster-wide and that DNS for `cloudbeaver.conderhome.xyz` points at the cluster's ingress IP.

### Can't Reach Target Databases

From inside the pod (or a debug pod in the same namespace), confirm DNS/network reachability to the target database Service:
```bash
kubectl exec -n tools deploy/cloudbeaver -- getent hosts shared-postgres-rw.<namespace>.svc.cluster.local
```
If this fails, the issue is cluster networking/DNS or a NetworkPolicy, not CloudBeaver itself.

## Scaling

CloudBeaver is not designed to scale horizontally in this deployment: the workspace PVC is `ReadWriteOnce` and the Deployment strategy is `Recreate`, both of which assume a single replica. If multi-replica access becomes necessary, it would require moving the workspace to a `ReadWriteMany`-capable storage class and validating that CloudBeaver CE supports concurrent backend instances sharing one workspace (not guaranteed for the Community Edition) — treat this as a non-trivial change, not a simple `replicas:` bump.

---

*This deployment guide is part of the CloudBeaver homelab deployment docs. For architecture rationale, see architecture.md; for day-to-day usage, see user-guide.md.*
