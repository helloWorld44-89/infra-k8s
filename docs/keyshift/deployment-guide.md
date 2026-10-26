# KeyShift Deployment Guide

## Prerequisites

- Kubernetes cluster (k3s v1.35+)
- Traefik ingress with wildcard TLS
- local-path storage provisioner
- kubectl / Flux access

## Deploy via Flux (GitOps)

```bash
cd /home/jconder44/homelab/infra-k8s
git add apps/tools/keyshift/
git commit -m "Deploy KeyShift"
git push
# Flux reconciles automatically
```

## Manual Deploy

```bash
kubectl apply -k apps/tools/keyshift/
```

## Configuration

### HelmRelease (`apps/tools/keyshift/helmrelease.yml`)

```yaml
image:
  repository: ghcr.io/helloworld44-89/keyshift
  tag: latest

env:
  FLASK_DEBUG: "false"
  DATABASE_URL: "sqlite:///KeyShift.db"
  SESSION_COOKIE_SECURE: "true"
  SESSION_COOKIE_HTTPONLY: "true"
  SESSION_COOKIE_SAMESITE: "Lax"
  TZ: "America/Chicago"
  FLASK_ENV: "production"
  SECRET_KEY:
    valueFrom:
      secretKeyRef:
        name: keyshift-secrets
        key: SECRET_KEY

persistence:
  config:
    storageClass: local-path
    size: 1Gi
    mountPath: /app/app/config/file
  db:
    storageClass: local-path
    size: 1Gi
    mountPath: /app/instance

resources:
  requests:
    cpu: 20m
    memory: 64Mi
  limits:
    cpu: 500m
    memory: 256Mi

probes:
  liveness:
    path: /healthz  # Rate-limit exempt
  readiness:
    path: /healthz

ingress:
  host: keyshift.conderhome.xyz
  tls: wildcard-conderhome-xyz-tls
```

### Required Secrets

Create before deploy:
```bash
kubectl create secret generic keyshift-secrets -n tools \
  --from-literal=SECRET_KEY="<your-secret-key>" \
  --dry-run=client -o yaml | kubectl apply -f -
```

### PVCs

- `keyshift-config` (1Gi, local-path) → `/app/app/config/file`
- `keyshift-db` (1Gi, local-path) → `/app/instance`

## Post-Deploy Verification

```bash
kubectl get pods -n tools -l app.kubernetes.io/instance=keyshift
kubectl logs -n tools -l app.kubernetes.io/instance=keyshift --tail=20
curl -k https://keyshift.conderhome.xyz/healthz
# Should return: {"status":"ok"}
```

## Migration from Docker (Historical)

Previous: Docker on 192.168.5.43
1. `docker stop keyshift`
2. Tar config (`/opt/docker/keyshift/data/config`) + db (`/opt/docker/keyshift/data/db`)
3. Copy to k3s node, extract into PVCs via node-debugger
4. Scale deployment to 1
5. Verify: login, config loads, `/healthz` returns ok
6. DNS: `keyshift.conderhome.xyz` → 192.168.5.65
7. Stop old Docker container

## Backup

```bash
# Config + DB
kubectl exec -n tools <pod> -- tar czf /tmp/ks-backup.tar.gz \
  /app/app/config/file /app/instance
kubectl cp tools/<pod>:/tmp/ks-backup.tar.gz ./ks-backup-$(date +%F).tar.gz
```

## Upgrade

1. Update image tag in helmrelease.yml
2. `git commit -m "Upgrade KeyShift to <tag>"`
3. `git push`
4. Flux rolls out new pod
5. Verify `/healthz` and login

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| CrashLoop (429) | Probes hit `/` rate limit | Ensure probes use `/healthz` |
| OOMKilled | 4 gunicorn workers > 256Mi | Increase memory limit to 512Mi |
| Config not loading | PVC mount path wrong | Check `advancedMounts` paths |
| Secret missing | `keyshift-secrets` not created | Create secret before deploy |

## Monitoring

- **Health**: `/healthz` (liveness/readiness)
- **Logs**: Flask/Gunicorn stdout
- **Metrics**: None exposed
- **Alerts**: Pod crashloop, PVC full