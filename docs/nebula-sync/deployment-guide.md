# Nebula Sync Deployment Guide

## Prerequisites

- Kubernetes cluster (k3s v1.35+)
- Two Pi-hole instances with API access
- kubectl / Flux access

## Deploy via Flux (GitOps)

```bash
cd /home/jconder44/homelab/infra-k8s
git add apps/tools/nebula-sync/
git commit -m "Deploy Nebula Sync"
git push
```

## Manual Deploy

```bash
kubectl apply -k apps/tools/nebula-sync/
```

## Configuration

### HelmRelease (`apps/tools/nebula-sync/helmrelease.yml`)

```yaml
image:
  repository: ghcr.io/helloworld44-89/nebula-sync
  tag: latest

env:
  PRIMARY_PIHOLE_URL: "https://192.168.5.5"
  SECONDARY_PIHOLE_URL: "https://192.168.5.6"

# Secrets referenced from nebula-sync-secrets
envFrom:
  - secretRef:
      name: nebula-sync-secrets

schedule: "0 * * * *"  # Hourly at minute 0

resources:
  requests:
    cpu: 10m
    memory: 32Mi
  limits:
    cpu: 100m
    memory: 128Mi
```

### Required Secret

Create before deploy (never commit to git):

```bash
kubectl create secret generic nebula-sync-secrets -n tools \
  --from-literal=PRIMARY_PIHOLE_API_KEY="<primary-api-key>" \
  --from-literal=SECONDARY_PIHOLE_API_KEY="<secondary-api-key>" \
  --dry-run=client -o yaml | kubectl apply -f -
```

Get API keys from each Pi-hole: Settings → API → Generate/Show API Key.

### CronJob Schedule

- Default: `"0 * * * *"` (hourly at minute 0)
- Override via HelmRelease `schedule` field

## Post-Deploy Verification

```bash
# Check CronJob
kubectl get cronjob nebula-sync -n tools

# Check recent jobs
kubectl get jobs -n tools -l app.kubernetes.io/instance=nebula-sync

# Check logs
kubectl logs -n tools -l app.kubernetes.io/instance=nebula-sync --tail=20
```

Expected log output:
```
INFO: Fetched 34 records from primary
INFO: Secondary has 34 records
INFO: No changes needed
```

## Migration from Docker (Historical)

Previous: Docker on 192.168.5.42
1. `docker stop nebula-sync`
2. No persistent data to migrate (stateless)
3. Deploy to k3s with same secrets
4. Verify sync runs
5. Stop old Docker container

## Upgrade

1. Update image tag in helmrelease.yml
2. `git commit -m "Upgrade Nebula Sync to <tag>"`
3. `git push`
4. Flux rolls out new CronJob
5. Next scheduled run uses new image

## Troubleshooting

| Symptom | Fix |
|---|---|
| CronJob not creating jobs | Check schedule syntax, suspend field |
| Jobs failing | Check logs for API errors |
| Auth failed | Verify API keys in secret |
| Secondary not updated | Check secondary Pi-hole API access |
| FTL not reloading | Sync script may need `pihole restartdns reload` |

## Monitoring

- **CronJob**: `kubectl get cronjob nebula-sync -n tools`
- **Jobs**: `kubectl get jobs -n tools -l app.kubernetes.io/instance=nebula-sync`
- **Logs**: `kubectl logs -n tools -l app.kubernetes.io/instance=nebula-sync`
- **Alerts**: Job failure (exit code != 0)

## Secrets Rotation

```bash
# Update secret
kubectl create secret generic nebula-sync-secrets -n tools \
  --from-literal=PRIMARY_PIHOLE_API_KEY="<new-key>" \
  --from-literal=SECONDARY_PIHOLE_API_KEY="<new-key>" \
  --dry-run=client -o yaml | kubectl replace -f -

# Next run uses new keys
```

## Monitoring

- **CronJob status**: `kubectl get cronjob`
- **Job history**: `kubectl get jobs -l app.kubernetes.io/instance=nebula-sync`
- **Logs**: Structured JSON for parsing
- **Alerts**: Job failure rate