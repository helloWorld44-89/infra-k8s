# NexusTale Deployment Guide

## Prerequisites

- Kubernetes cluster (k3s v1.35+)
- Traefik ingress with wildcard TLS
- CNPG (CloudNative-PG) operator installed
- Redis deployment
- kubectl / Flux access

## Deploy via Flux (GitOps)

```bash
cd /home/jconder44/homelab/infra-k8s
git add apps/tools/nexustale/
git commit -m "Deploy NexusTale"
git push
```

## Manual Deploy

```bash
kubectl apply -k apps/tools/nexustale/
```

## Configuration

### Prerequisites (Deploy First)

1. **CNPG Cluster** (`shared-postgres` in `databases` namespace)
2. **Redis** (`nexustale-redis` deployment + PVC)
3. **Secrets** (`nexustale-secrets` in `tools` namespace)

### Required Secrets

```bash
kubectl create secret generic nexustale-secrets -n tools \
  --from-literal=JWT_SECRET_KEY="<64-char-random>" \
  --from-literal=DATABASE_PASSWORD="<db-password>" \
  --from-literal=REDIS_PASSWORD="<redis-password>" \
  --from-literal=SECRET_KEY="<django-secret>" \
  --dry-run=client -o yaml | kubectl apply -f -
```

### CNPG Database

```yaml
# In databases namespace, part of shared-postgres cluster
apiVersion: postgresql.cnpg.io/v1
kind: Database
metadata:
  name: nexustale
  namespace: databases
spec:
  name: nexustale
  owner: nexustale
  extensions:
    - name: vector  # pgvector
```

### HelmRelease (`apps/tools/nexustale/helmrelease.yml`)

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: nexustale
  namespace: tools
spec:
  chart:
    spec:
      chart: app-template
      version: "3.x"
      sourceRef:
        kind: HelmRepository
        name: bjw-s
        namespace: flux-system
  values:
    # API Deployment
    api:
      image:
        repository: ghcr.io/helloworld44-89/nexustale/api
        tag: v0.3.2
      env:
        DATABASE_URL: "postgresql://nexustale:${DATABASE_PASSWORD}@shared-postgres-rw.databases:5432/nexustale"
        REDIS_URL: "redis://:${REDIS_PASSWORD}@nexustale-redis.tools:6379/0"
        JWT_SECRET_KEY: "${JWT_SECRET_KEY}"
        SECRET_KEY: "${SECRET_KEY}"
        CORS_ORIGINS: "https://nexustale.conderhome.xyz"
      resources:
        requests: {cpu: 100m, memory: 256Mi}
        limits: {cpu: 1000m, memory: 1Gi}
      probes:
        liveness: {path: /health, port: 8000}
        readiness: {path: /health, port: 8000}

    # Frontend Deployment
    frontend:
      image:
        repository: ghcr.io/helloworld44-89/nexustale/frontend
        tag: v0.3.2
      resources:
        requests: {cpu: 20m, memory: 64Mi}
        limits: {cpu: 200m, memory: 256Mi}

    # Redis
    redis:
      enabled: true
      image: redis:7-alpine
      persistence:
        enabled: true
        storageClass: local-path
        size: 1Gi

    # Ingress
    ingress:
      api:
        host: nexustale.conderhome.xyz
        path: /api
        tls: wildcard-conderhome-xyz-tls
      frontend:
        host: nexustale.conderhome.xyz
        path: /
        tls: wildcard-conderhome-xyz-tls
```

## Post-Deploy Verification

```bash
# Check all pods
kubectl get pods -n tools -l app.kubernetes.io/instance=nexustale

# Check API health
curl -k https://nexustale.conderhome.xyz/api/health

# Check frontend
curl -k https://nexustale.conderhome.xyz/

# Check DB migrations
kubectl logs -n tools -l app=nexustale-api --tail=30 | grep -i migration
```

Expected: `Migrations: DONE` (or Alembic output showing v40)

## Migration from Docker (Historical)

Previous: Docker Compose on 192.168.5.43 (5 containers)
Issues: v0.3.1 had MinIO dependency, broken migration 37
Fix: v0.3.2 removed MinIO, pgvector pre-installed

1. Stop Docker: `docker compose down`
2. Export DB: `docker exec nexustale-postgres pg_dump -U nexustale nexustale > dump.sql`
3. Deploy k3s with v0.3.2 (clean DB)
4. Import: `kubectl exec -i nexustale-postgres-0 -n databases -- psql -U nexustale nexustale < dump.sql`
5. Verify: Login, check notes, search works
6. DNS: `nexustale.conderhome.xyz` → 192.168.5.65
7. Stop Docker containers

## Backup

```bash
# PostgreSQL (via CNPG)
kubectl exec -n databases shared-postgres-0 -- pg_dump -U nexustale nexustale | gzip > nexustale-db-$(date +%F).sql.gz

# Redis (if needed)
kubectl exec -n tools nexustale-redis-0 -- redis-cli --rdb /tmp/redis.rdb
kubectl cp tools/nexustale-redis-0:/tmp/redis.rdb ./redis-$(date +%F).rdb
```

## Upgrade

1. Update image tags in helmrelease.yml
2. `git commit -m "Upgrade NexusTale to v0.x.x"`
3. `git push`
4. Flux rolls out API + Frontend
5. Verify: `/api/health`, login, search works

## Troubleshooting

| Symptom | Fix |
|---|---|
| Migration 37 fails | Ensure v0.3.2 (pgvector pre-installed), clean DB |
| API can't connect to DB | Check `DATABASE_URL`, CNPG service name, credentials |
| Redis connection refused | Check Redis pod, password in secret, network policy |
| CORS errors | Verify `CORS_ORIGINS` matches frontend URL |
| Search not working | Check pgvector extension, re-index embeddings |

## Monitoring

- **API**: `/health` (liveness/readiness)
- **Frontend**: nginx status
- **DB**: CNPG cluster status (`kubectl get cluster -n databases`)
- **Redis**: `redis-cli ping`
- **Alerts**: Pod crashloop, PVC full, DB connection failures