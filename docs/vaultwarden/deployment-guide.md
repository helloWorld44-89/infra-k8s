# Vaultwarden Deployment Guide

## Prerequisites

- Kubernetes cluster (k3s v1.35+)
- Traefik ingress controller with wildcard TLS cert
- local-path storage provisioner
- Helm 3.x / Flux CD
- kubectl access (cluster-admin)

## Quick Deploy

```bash
# Via Flux (GitOps - recommended)
cd /home/jconder44/homelab/infra-k8s
git add apps/tools/vaultwarden/
git commit -m "Deploy Vaultwarden"
git push
# Flux reconciles automatically
```

## Manual Deploy

```bash
# Apply kustomization directly
kubectl apply -k apps/tools/vaultwarden/
```

## Configuration

### HelmRelease (`apps/tools/vaultwarden/helmrelease.yml`)

```yaml
image:
  repository: vaultwarden/server
  tag: "1.37.4"  # Pinned - do not use :latest

env:
  DOMAIN: "https://vault.conderhome.xyz"
  ROCKET_PROFILE: "release"
  ROCKET_ADDRESS: "0.0.0.0"
  ROCKET_PORT: "80"
  TZ: "America/Chicago"

persistence:
  data:
    storageClass: local-path
    size: 1Gi
    accessMode: ReadWriteOnce

resources:
  requests:
    cpu: 20m
    memory: 64Mi
  limits:
    cpu: 500m
    memory: 256Mi

ingress:
  className: traefik
  hosts:
    - host: vault.conderhome.xyz
      paths:
        - path: /
          pathType: Prefix
  tls:
    - hosts: [vault.conderhome.xyz]
      secretName: wildcard-conderhome-xyz-tls
```

### Required Secrets

None - Vaultwarden manages its own encryption keys in the SQLite database.

### Ingress TLS

Uses pre-existing wildcard certificate:
- Secret: `wildcard-conderhome-xyz-tls` (mirrored to `tools` namespace via reflector)
- Issued by: cert-manager + Cloudflare DNS-01

## Post-Deploy Verification

```bash
# Check pod
kubectl get pods -n tools -l app.kubernetes.io/instance=vaultwarden

# Check logs
kubectl logs -n tools -l app.kubernetes.io/instance=vaultwarden --tail=20

# Health check
curl -k https://vault.conderhome.xyz/alive

# Full site
curl -k https://vault.conderhome.xyz/
```

Expected: HTTP 200, login page loads.

## Migration from Docker (Historical)

Previous deployment: Docker on 192.168.5.43
1. `docker stop vaultwarden` (graceful shutdown - checkpoints WAL)
2. `docker run --rm -v /opt/docker/vaultwarden/vw-data:/data:ro -v /tmp:/backup alpine tar czf /backup/vw.tar.gz -C /data .`
3. Copy to k3s node, extract into PVC via node-debugger pod
4. `kubectl scale deployment vaultwarden -n tools --replicas=1`
5. Verify: `PRAGMA integrity_check` on restored DB, login test
6. DNS update: `vault.conderhome.xyz` → 192.168.5.65 (Pi-hole)
7. `docker stop vaultwarden` on old host

## Backup Procedure

```bash
# Logical backup (recommended - consistent)
kubectl exec -n tools <vaultwarden-pod> -- sqlite3 /data/db.sqlite3 ".backup /tmp/vw-backup.sqlite3"
kubectl cp tools/<pod>:/tmp/vw-backup.sqlite3 ./vw-backup-$(date +%F).sqlite3

# Or PVC snapshot (if storage class supports)
# kubectl create -f vaultwarden-pvc-snapshot.yaml
```

## Restore Procedure

```bash
# Scale to 0
kubectl scale deployment vaultwarden -n tools --replicas=0

# Restore PVC via node-debugger (see migration notes)
# Then scale back
kubectl scale deployment vaultwarden -n tools --replicas=1
```

## Upgrade Procedure

1. Update `image.tag` in `helmrelease.yml` to new version
2. `git commit -m "Upgrade Vaultwarden to <version>"`
3. `git push`
4. Flux reconciles, rolls out new pod
5. Verify health checks pass

**Never use `:latest` tag** - pin exact version (e.g., `1.37.4`)

## Troubleshooting

| Symptom | Check |
|---|---|
| Pod CrashLoop | `kubectl logs -n tools -l app.kubernetes.io/instance=vaultwarden --previous` |
| 502 Bad Gateway | Traefik can't reach pod - check service/endpoints, pod readiness |
| TLS errors | Verify `wildcard-conderhome-xyz-tls` secret exists in `tools` namespace |
| DB locked | Only one replica allowed (SQLite), check for duplicate pods |
| Slow performance | Check PVC latency (`iostat` on node), consider larger PVC |

## Security Hardening

- [ ] Admin token not configured (admin panel disabled)
- [ ] TLS enforced at ingress (no HTTP)
- [ ] No external access to SQLite file
- [ ] Regular logical backups
- [ ] Monitor for CVEs in base image
- [ ] Update image tag promptly on security releases

## Monitoring

- **Health**: `/alive` endpoint (liveness/readiness probes)
- **Metrics**: None exposed natively (consider Prometheus exporter sidecar if needed)
- **Logs**: Standard stdout/stderr via kubectl
- **Alerts**: Pod crashloop, PVC full, cert expiry