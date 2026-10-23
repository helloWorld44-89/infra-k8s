---
title: MySQL Deployment Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [deployment, kubernetes, flux, database]
---

# MySQL Deployment Guide

## Overview

MySQL is deployed into the homelab k3s cluster as a Flux-managed `HelmRelease` using the `bjw-s` `app-template` chart. This guide covers the existing GitOps deployment, how to change its configuration, and operational tasks (backup/restore, scaling considerations, troubleshooting) specific to this instance.

## Prerequisites

### Cluster Requirements

- A working k3s cluster with Flux v2 installed (`helm.toolkit.fluxcd.io/v2` CRDs)
- The `bjw-s` Helm repository already registered as a Flux `HelmRepository` named `bjw-s` in the `flux-system` namespace
- An `nfs` `StorageClass` available for `ReadWriteOnce` PVCs
- `kubectl` access with permissions in the `tools` namespace

### Repository Layout

```
apps/tools/mysql/
├── kustomization.yaml   # bundles the HelmRelease + secrets
├── helmrelease.yml       # the HelmRelease spec (chart values)
└── secrets.yml           # Secret manifest (mysql-secrets) — typically SOPS/SealedSecrets-encrypted
```

This is referenced from `apps/tools/kustomization.yaml`, which lists `mysql` alongside `docs-site` and `homepage`.

## Deployment Method: Flux GitOps (Current / Only Method)

This cluster deploys everything through Flux — there is no manual `helm install` or `docker run` path in normal operation. All changes are made by editing files in Git and letting Flux reconcile them.

### 1. HelmRelease Definition

`apps/tools/mysql/helmrelease.yml`:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: mysql
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
      mysql:
        containers:
          app:
            image:
              repository: mysql
              tag: "8"
            env:
              MYSQL_DATABASE: my_database
              MYSQL_ROOT_PASSWORD:
                valueFrom:
                  secretKeyRef:
                    name: mysql-secrets
                    key: MYSQL_ROOT_PASSWORD
              MYSQL_USER:
                valueFrom:
                  secretKeyRef:
                    name: mysql-secrets
                    key: MYSQL_USER
              MYSQL_PASSWORD:
                valueFrom:
                  secretKeyRef:
                    name: mysql-secrets
                    key: MYSQL_PASSWORD
    service:
      app:
        controller: mysql
        ports:
          http:
            port: 3306
    persistence:
      data:
        accessMode: ReadWriteOnce
        size: 1Gi
        globalMounts:
          - path: /var/lib/mysql
```

### 2. Secret Definition

`apps/tools/mysql/secrets.yml` holds the `mysql-secrets` Secret referenced above (`MYSQL_ROOT_PASSWORD`, `MYSQL_USER`, `MYSQL_PASSWORD`). Treat this file as sensitive — it should be encrypted at rest in Git (SOPS/SealedSecrets per this repo's existing convention) rather than committed in plaintext.

### 3. Kustomization

`apps/tools/mysql/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - helmrelease.yml
  - secrets.yml
```

### 4. Apply via Git

```bash
git add apps/tools/mysql/
git commit -m "tools/mysql: <describe change>"
git push
```

Flux reconciles on its normal interval, or force it immediately:

```bash
flux reconcile helmrelease mysql -n tools --with-source
```

### 5. Verify Rollout

```bash
# Check HelmRelease status
kubectl get helmrelease mysql -n tools

# Check pod
kubectl get pods -n tools -l app.kubernetes.io/name=mysql

# Check logs for successful startup
kubectl logs -n tools deploy/mysql
```

Expect a log line similar to `mysqld: ready for connections` once the container has finished InnoDB initialization.

## Configuration

### Environment Variables

| Variable | Source | Description |
|----------|--------|--------------|
| `MYSQL_DATABASE` | Inline value in `helmrelease.yml` | Database created automatically on first boot (`my_database`) |
| `MYSQL_ROOT_PASSWORD` | `mysql-secrets` Secret | Root account password |
| `MYSQL_USER` | `mysql-secrets` Secret | Non-root application username |
| `MYSQL_PASSWORD` | `mysql-secrets` Secret | Password for `MYSQL_USER` |

To change the default database name or add additional init SQL, this would require extending the chart values (e.g. mounting an `/docker-entrypoint-initdb.d` ConfigMap) — not currently configured.

### Volume Mounts

| Path | Purpose | Backing |
|------|---------|---------|
| `/var/lib/mysql` | Full MySQL data directory (InnoDB tablespaces, grants, binlogs) | PVC `mysql-data`, `nfs` StorageClass, 1Gi, RWO |

### Networking

| Setting | Value |
|---------|-------|
| Service name | `mysql` |
| Namespace | `tools` |
| Port | `3306` (named `http` in chart values — a labeling convention of the `app-template` chart, not an actual HTTP endpoint) |
| Type | `ClusterIP` (in-cluster only) |
| Ingress | None defined |

## Operational Tasks

### Scaling Storage

The PVC is fixed at `1Gi`. To grow it (assuming the underlying StorageClass supports expansion):

```bash
kubectl patch pvc mysql-data -n tools -p '{"spec":{"resources":{"requests":{"storage":"5Gi"}}}}'
```

Then update `persistence.data.size` in `helmrelease.yml` to match, so Git stays the source of truth.

### Restarting the Database

```bash
kubectl rollout restart deployment/mysql -n tools
```

Because the deployment strategy is `Recreate`, the existing pod terminates fully before the replacement starts (expected brief downtime, acceptable for a single-replica database).

### Backup

```bash
# Logical backup (recommended for portability)
kubectl exec -n tools deploy/mysql -- \
  sh -c 'mysqldump -uroot -p"$MYSQL_ROOT_PASSWORD" my_database' > my_database-$(date +%Y%m%d).sql

# Compress
gzip my_database-$(date +%Y%m%d).sql
```

### Restore

```bash
gunzip my_database-20261023.sql.gz

kubectl exec -i -n tools deploy/mysql -- \
  sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" my_database' < my_database-20261023.sql
```

### Rotating Credentials

1. Update the values in `mysql-secrets` (re-encrypt with SOPS/SealedSecrets per repo convention, then commit)
2. Note: MySQL only re-reads `MYSQL_ROOT_PASSWORD`/`MYSQL_USER`/`MYSQL_PASSWORD` on **first initialization** of an empty data directory — changing the Secret alone does **not** change an already-initialized database's passwords
3. To actually apply a new password to an existing database, connect as root and run `ALTER USER` / `SET PASSWORD` manually, then update the Secret to match:
   ```sql
   ALTER USER '<MYSQL_USER>'@'%' IDENTIFIED BY '<new_password>';
   ALTER USER 'root'@'%' IDENTIFIED BY '<new_root_password>';
   FLUSH PRIVILEGES;
   ```

## Troubleshooting

### Pod Won't Start

```bash
kubectl describe pod -n tools -l app.kubernetes.io/name=mysql
kubectl logs -n tools deploy/mysql --previous
```

Common causes: PVC stuck in `Pending` (check the `nfs` StorageClass/NFS server availability), or a malformed/missing `mysql-secrets` key causing `CreateContainerConfigError`.

### CrashLoopBackOff After a Config Change

Data directory corruption or an incompatible `MYSQL_*` env change against an already-initialized volume is the most common cause — the `mysql:8` entrypoint only applies `MYSQL_DATABASE`/`MYSQL_USER`/`MYSQL_PASSWORD` on a **fresh, empty** `/var/lib/mysql`. Check logs for `InnoDB: Database was not shutdown normally` or similar.

### Checking Disk Usage

```bash
kubectl exec -n tools deploy/mysql -- df -h /var/lib/mysql
kubectl exec -n tools deploy/mysql -- sh -c 'du -sh /var/lib/mysql/*'
```

### Connectivity Issues from Other Pods

```bash
kubectl run -n tools mysql-test --rm -it --image=mysql:8 --restart=Never -- \
  mysql -h mysql.tools.svc.cluster.local -P 3306 -u<MYSQL_USER> -p
```

## Migration / Version Upgrades

The image tag is pinned to `mysql:8` (major-version float, not a specific point release). To pin to an exact version for reproducibility:

```yaml
image:
  repository: mysql
  tag: "8.4.3"   # example — check current mysql:8 minor for the actual version running
```

Before any major version bump (e.g. MySQL 8 → 9), take a full `mysqldump` backup first — in-place major upgrades on a single-replica, non-HA database carry real risk of an unrecoverable data directory if interrupted.

---

*This deployment guide covers the Flux-managed `mysql` HelmRelease in the `tools` namespace of the homelab k3s cluster. All changes should go through Git/Flux — avoid manual, untracked `kubectl edit`/`helm upgrade` changes to this resource.*
