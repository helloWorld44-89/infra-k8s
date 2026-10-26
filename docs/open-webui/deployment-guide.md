# Open WebUI Deployment Guide

## Prerequisites

- Kubernetes cluster (k3s v1.35+)
- Traefik ingress with wildcard TLS
- local-path storage provisioner
- kubectl / Flux access
- Optional: Ollama deployment for local LLMs

## Deploy via Flux (GitOps)

```bash
cd /home/jconder44/homelab/infra-k8s
git add apps/tools/open-webui/
git commit -m "Deploy Open WebUI"
git push
```

## Manual Deploy

```bash
kubectl apply -k apps/tools/open-webui/
```

## Configuration

### HelmRelease (`apps/tools/open-webui/helmrelease.yml`)

```yaml
image:
  repository: ghcr.io/open-webui/open-webui
  tag: latest

env:
  OLLAMA_BASE_URL: "http://ollama:11434"  # Optional
  WEBUI_SECRET_KEY: ""  # Auto-generated if empty
  WEBUI_AUTH: "True"  # Enable auth
  DEFAULT_MODELS: ""  # Comma-separated model IDs
  ENABLE_SIGNUP: "True"
  DEFAULT_USER_ROLE: "user"

persistence:
  data:
    storageClass: local-path
    size: 10Gi  # Vector DB + uploads can grow
    accessMode: ReadWriteOnce

resources:
  requests:
    cpu: 100m
    memory: 512Mi
  limits:
    cpu: 2000m
    memory: 4Gi

probes:
  liveness:
    path: /health
    initialDelaySeconds: 30
  readiness:
    path: /health
    initialDelaySeconds: 30

ingress:
  host: ai.conderhome.xyz
  tls: wildcard-conderhome-xyz-tls
```

### PVC

- `open-webui-data` (10Gi, local-path) → `/app/backend/data`
- Contains: `webui.db`, `vector_db/`, `uploads/`, `cache/`

## Post-Deploy Verification

```bash
kubectl get pods -n tools -l app.kubernetes.io/instance=open-webui
kubectl logs -n tools -l app.kubernetes.io/instance=open-webui --tail=30
curl -k https://ai.conderhome.xyz/health
# {"status":"ok"}
```

## Migration from Docker (Historical)

Previous: Docker on 192.168.5.43
1. `docker stop openwebui`
2. Tar `/opt/docker/openwebui` (805MB)
3. Copy to prod-k3s-1, extract into PVC via node-debugger
4. Scale deployment to 1
4. Verify: `webui.db` present, embeddings load, login works
5. DNS: `ai.conderhome.xyz` → 192.168.5.65 (Pi-hole)
6. Stop old Docker container

## Backup

```bash
# Full data (includes vector DB - can be large)
kubectl exec -n tools <pod> -- tar czf /tmp/owui-backup.tar.gz -C /app/backend/data .
kubectl cp tools/<pod>:/tmp/owui-backup.tar.gz ./owui-backup-$(date +%F).tar.gz
```

## Upgrade

1. Update image tag in helmrelease.yml
2. `git commit -m "Upgrade Open WebUI to <tag>"`
3. `git push`
4. Flux rolls out
5. Verify: models load, chats work, RAG indexes intact

## Troubleshooting

| Symptom | Fix |
|---|---|
| Pod OOMKilled | Increase memory limit (vector DB + model cache) |
| Model not found | Pull model in UI, check Ollama connectivity |
| RAG not working | Check vector DB permissions, re-index |
| 502 after upgrade | Clear browser cache, check pod readiness |
| Lost admin | Delete `webui.db`, restart pod |

## Migration Notes

- `webui.db` contains users, chats, settings, API keys
- Vector DB (`vector_db/`) stores embeddings - large but rebuildable
- Uploads (`uploads/`) - user files
- Cache (`cache/`) - model cache, safe to delete

## Monitoring

- **Health**: `/health` (liveness/readiness)
- **Logs**: FastAPI/Uvicorn stdout
- **Metrics**: None exposed
- **Alerts**: Pod crashloop, PVC full, OOMKilled