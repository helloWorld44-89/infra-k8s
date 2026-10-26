# OpusMaxim Deployment Guide

## Prerequisites

- Kubernetes cluster (k3s v1.35+)
- Traefik ingress with wildcard TLS
- local-path storage provisioner
- kubectl / Flux access

## Deploy via Flux (GitOps)

```bash
cd /home/jconder44/homelab/infra-k8s
git add apps/tools/opusmaxim/
git commit -m "Deploy OpusMaxim"
git push
```

## Manual Deploy

```bash
kubectl apply -k apps/tools/opusmaxim/
```

## Configuration

### HelmRelease (`apps/tools/opusmaxim/helmrelease.yml`)

```yaml
image:
  repository: ghcr.io/helloworld44-89/opusmaxim
  tag: latest

env:
  FLASK_DEBUG: "false"
  SECRET_KEY:
    valueFrom:
      secretKeyRef:
        name: opusmaxim-secrets
        key: SECRET_KEY
  TZ: "America/Chicago"

persistence:
  data:
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
    path: /
  readiness:
    path: /

ingress:
  host: opusmaxim.conderhome.xyz
  tls: wildcard-conderhome-xyz-tls
```

### Required Secret

```bash
kubectl create secret generic opusmaxim-secrets -n tools \
  --from-literal=SECRET_KEY="<your-secret-key>" \
  --dry-run=client -o yaml | kubectl apply -f -
```

### PVC

- `opusmaxim-data` (1Gi, local-path) → `/app/instance`
- Contains: `opus.db` (SQLite), `avatars/`

## Post-Deploy Verification

```bash
kubectl get pods -n tools -l app.kubernetes.io/instance=opusmaxim
kubectl logs -n tools -l app.kubernetes.io/instance=opusmaxim --tail=20
curl -k https://opusmaxim.conderhome.xyz/
# Should load homepage (200)
```

## Migration from Docker (Historical)

Previous: Docker on 192.168.5.43
1. `docker stop opusmaxim`
2. Tar `/opt/docker/opusmaxim` (460KB DB + 88K avatars)
3. Copy to k3s node, extract into PVC via node-debugger
4. Scale deployment to 1
4. Verify: login works, data intact
5. DNS: `opusmaxim.conderhome.xyz` → 192.168.5.65
6. Stop old Docker container

## Backup

```bash
kubectl exec -n tools <pod> -- tar czf /tmp/opus-backup.tar.gz -C /app/instance .
kubectl cp tools/<pod>:/tmp/opus-backup.tar.gz ./opus-backup-$(date +%F).tar.gz
```

## Upgrade

1. Update image tag in helmrelease.yml
2. `git commit -m "Upgrade OpusMaxim to <tag>"`
3. `git push`
4. Flux rolls out
4. Verify login and data

## Troubleshooting

| Symptom | Fix |
|---|---|
| Pod CrashLoop | Check logs for DB errors |
| 502 Bad Gateway | Pod not ready, check readiness probe |
| DB locked | Only one replica, check for duplicate pods |
| Static files 404 | Check Flask static folder config |

## Monitoring

- **Health**: `/` (liveness/readiness)
- **Logs**: Flask/Gunicorn stdout
- **Metrics**: None exposed
- **Alerts**: Pod crashloop, PVC full