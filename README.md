# Homelab K8s Infrastructure

## Cluster Overview

| Node | Role | IP |
|------|------|----|
| prod-k3s-master | control-plane | 192.168.5.x |
| prod-k3s-1 | worker | 192.168.5.x |
| prod-k3s-2 | worker | 192.168.5.x |
| prod-k3s-3 | worker | 192.168.5.x |

## NFS Storage
- **Server:** 192.168.5.18
- **Path:** /mnt/ConderNAS/k3s
- **Media Path:** /mnt/ConderNAS/media
- **Downloads Path:** /mnt/ConderNAS/downloads
- **StorageClass:** nfs (default)

## Ingress
- **Controller:** Traefik (installed via K3s)

## GitOps
- Flux CD watches this repo on the `main` branch
- Infrastructure is deployed before apps via `dependsOn`
- `prune: true` means resources deleted from git are deleted from the cluster

## Useful Commands
```bash
# Check flux sync status
flux get all

# Force a sync
flux reconcile kustomization apps

# Watch pods
kubectl get pods -A -w

# Check events
kubectl get events -A --sort-by=.lastTimestamp | tail -20
```

## Secrets
- Never commit plain Secret manifests
- Use kubeseal to encrypt: `kubeseal --format yaml < secret.yaml > sealed-secret.yaml`
- Sealed secrets are safe to commit
