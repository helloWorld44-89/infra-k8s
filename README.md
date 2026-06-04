# Homelab Kubernetes Infrastructure

![Flux CD](https://img.shields.io/badge/GitOps-Flux_CD_v2-blue?logo=flux)
![K3s](https://img.shields.io/badge/Kubernetes-K3s-orange?logo=kubernetes)
![SOPS](https://img.shields.io/badge/Secrets-SOPS_%2B_Age-green?logo=gnuprivacyguard)
![Validate](https://github.com/helloWorld44-89/infra-k8s/actions/workflows/validate.yaml/badge.svg)

GitOps-managed homelab Kubernetes cluster using Flux CD. All cluster state lives in this repo — pushing to `main` is the only deployment mechanism.

## Architecture

```
┌─────────────┐     watches      ┌──────────────────────────────────────────┐
│   GitHub    │ ──────────────►  │              Flux CD (flux-system)        │
│  (this repo)│                  │                                           │
└─────────────┘                  │  Kustomization: infrastructure            │
                                 │    └─► namespaces, helm-repos, storage    │
                                 │                                           │
                                 │  Kustomization: apps (depends on infra)   │
                                 │    └─► monitoring, tools, jobops          │
                                 └──────────────┬───────────────────────────┘
                                                │ reconciles
                                                ▼
                          ┌─────────────────────────────────────────┐
                          │            K3s Cluster                  │
                          │                                         │
                          │  prod-k3s-master  192.168.5.x           │
                          │  prod-k3s-1       192.168.5.x  (worker) │
                          │  prod-k3s-2       192.168.5.x  (worker) │
                          │  prod-k3s-3       192.168.5.x  (worker) │
                          └─────────────────────────────────────────┘
```

## Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| Orchestration | [K3s](https://k3s.io/) | Lightweight Kubernetes distribution |
| GitOps | [Flux CD v2](https://fluxcd.io/) | Continuous reconciliation from git |
| Package Management | [Helm](https://helm.sh/) + [Kustomize](https://kustomize.io/) | Templating and overlays |
| Ingress | [Traefik](https://traefik.io/) | Reverse proxy (bundled with K3s) |
| Secrets | [SOPS](https://getsops.io/) + [Age](https://age-encryption.org/) | Encrypted secrets committed to git |
| Storage | NFS (via [nfs-subdir-external-provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)) | Persistent volumes backed by NAS |
| DNS | [external-dns](https://github.com/kubernetes-sigs/external-dns) + Pi-hole | Automatic DNS records from Ingress |
| Metrics | [Prometheus](https://prometheus.io/) | Cluster + node scraping |
| Dashboards | [Grafana](https://grafana.com/) | Visualization, custom homelab dashboard |
| Uptime | [Uptime Kuma](https://github.com/louislam/uptime-kuma) | Service health monitoring |

## Repository Structure

```
infra-k8s/
├── clusters/prod/          # Flux entry point — bootstraps everything
│   └── flux-system/        # Flux system components
├── infrastructure/         # Cluster-level resources (deployed first)
│   ├── helm-repos/         # HelmRepository sources
│   ├── namespaces/         # Namespace definitions
│   └── storage/            # NFS provisioner and StorageClass
└── apps/                   # Application workloads (depend on infrastructure)
    ├── jobops/             # Custom app — job tracking tool
    ├── monitoring/
    │   ├── grafana/        # Dashboards, Prometheus datasource
    │   ├── prometheus/     # Scrape configs for nodes and cluster
    │   └── uptimekuma/     # Service uptime monitoring
    └── tools/
        ├── cloudbeaver/    # Database management UI
        ├── external-dns/   # Auto-creates Pi-hole DNS records from Ingress
        ├── homepage/       # Internal service dashboard
        ├── kubernetes-dashboard/ # K8s web UI
        ├── mysql/          # MySQL 8 instance
        └── plane/          # Self-hosted project management (Postgres + Redis + MinIO)
```

## How GitOps Works

Flux runs inside the cluster and polls this repo every minute. On any change to `main`:

1. `infrastructure` Kustomization reconciles first (namespaces, Helm repos, NFS storage)
2. `apps` Kustomization reconciles after (`dependsOn: infrastructure` enforces this)
3. `prune: true` means resources deleted from git are automatically removed from the cluster

No `kubectl apply` or Helm commands needed for deployments — git is the source of truth.

## Secrets Management

Secrets are encrypted with [SOPS](https://getsops.io/) using an [Age](https://age-encryption.org/) key before being committed. Flux decrypts them at reconcile time using the `sops-age` Kubernetes secret.

```
# Encrypt a secret before committing
sops --encrypt --age <age-public-key> secret.yaml > secrets.yaml

# Decrypt locally for inspection (requires your Age private key)
sops --decrypt secrets.yaml
```

All files matching `secrets.ya?ml` are automatically encrypted per `.sops.yaml`. Plain secret files are `.gitignore`d as a safety net.

> The CI pipeline verifies that every `secrets.yaml` file in the repo contains the `sops:` metadata block — unencrypted secrets will fail the build.

## Storage

- **NFS Server:** 192.168.5.18 (`/mnt/ConderNAS/k3s`)
- **StorageClass:** `nfs` (default) — dynamically provisions subdirectories per PVC
- **Media volumes:** 500Gi NFS PVs for movies, TV, audiobooks, downloads

## Useful Commands

```bash
# Check sync status across all Flux resources
flux get all

# Force a reconciliation without waiting for the interval
flux reconcile kustomization apps --with-source

# Check HelmRelease status
flux get helmreleases -A

# Watch all pods across namespaces
kubectl get pods -A -w

# Tail recent cluster events
kubectl get events -A --sort-by=.lastTimestamp | tail -20

# Check SOPS decryption is working
flux logs --kind=Kustomization --name=apps -n flux-system
```

## Adding a New App

1. Create `apps/<name>/` with a `helmrelease.yml` and `kustomization.yaml`
2. Add `- <name>` to the appropriate `apps/tools/kustomization.yaml` or `apps/kustomization.yaml`
3. Encrypt any secrets: `sops --encrypt --age <key> secret.yaml > secrets.yaml`
4. Commit and push — Flux will reconcile within 1 minute
