# Actual Budget Deployment Guide

## Prerequisites

- Kubernetes cluster (k3s v1.35+)
- Traefik ingress with wildcard TLS
- local-path storage provisioner
- kubectl / Flux access

## Deploy via Flux (GitOps)

```bash
cd /home/jconder44/homelab/infra-k8s
git add apps/tools/actual-budget/
git commit -m "Deploy Actual Budget"
git push
```

## Manual Deploy

```bash
kubectl apply -k apps/tools/actual-budget/
```

## Configuration

### HelmRelease (`apps/tools/actual-budget/helmrelease.yml`)

```yaml
image:
  repository: actualbudget/actual-server
  tag: latest

env:
  ACTUAL_UPLOAD_FILE_SYNC_SIZE_LIMIT_MB: "20"
  NODE_ENV: "production"

persistence:
  data:
    storageClass: local-path
    size: 1Gi
    accessMode: ReadWriteOnce

resources:
  requests:
    cpu: 20m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

probes:
  liveness:
    path: /
  readiness:
    path: /

ingress:
  host: budget.conderhome.xyz
  tls: wildcard-conderhome-xyz-tls
```

### PVC

- `actual-budget-data` (1Gi, local-path) → `/data`

## Post-Deploy Verification

```bash
kubectl get pods -n tools -l app.kubernetes.io/instance=actual-budget
kubectl logs -n tools -l app.kubernetes.io/instance=actual-budget --tail=20
curl -k https://budget.conderhome.xyz/
# Should load React app (200)
```

## Migration from Docker (Historical)

Previous: Docker on 192.168.5.43 (`actual-actual_server-1-old`)
1. `docker stop actual-actual_server-1-old`
2. `docker run --rm -v /opt/docker/actual/actual-data:/data:ro -v /tmp:/backup alpine tar czf /backup/ab.tar.gz -C /data .`
3. Copy to k3s node, extract into PVC via node-debugger
4. Scale deployment to 1
5. Verify: migrations run (`Migrations: DONE`), login works
6. DNS: `budget.conderhome.xyz` → 192.168.5.65 (Pi-hole)
6. Stop old Docker container

## Backup

```bash
# Full data directory
kubectl exec -n tools <pod> -- tar czf /tmp/ab-backup.tar.gz -C /data .
kubectl cp tools/<pod>:/tmp/ab-backup.tar.gz ./ab-backup-$(date +%F).tar.gz
```

## Upgrade

1. Update image tag in helmrelease.yml
2. `git commit -m "Upgrade Actual Budget to <tag>"`
3. `git push`
4. Flux rolls out, runs migrations automatically
5. Verify login and data integrity

## Troubleshooting

| Symptom | Fix |
|---|---|
| Pod CrashLoop | Check logs for migration errors |
| 502 Bad Gateway | Pod not ready, check readiness probe |
| Data missing | Wrong PVC mounted, check mount path |
| Sync not working | Check user-files permissions, server URL |

## Backup

```bash
kubectl exec -n tools <pod> -- tar czf /tmp/ab-backup.tar.gz -C /data .
kubectl cp tools/<pod>:/tmp/ab-backup.tar.gz ./ab-backup-$(date +%F).tar.gz
```

## Monitoring

- **Health**: `/` (liveness/readiness)
- **Logs**: Node.js stdout
- **Metrics**: None exposed
- **Alerts**: Pod crashloop, PVC full