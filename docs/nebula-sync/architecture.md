# Nebula Sync Architecture

## Overview

Nebula Sync is a lightweight Pi-hole DNS synchronization tool that replicates Local DNS Records (Custom.list) from a primary Pi-hole to secondary instances. Stateless, cron-driven.

## Components

| Component | Technology | Purpose |
|---|---|---|
| `nebula-sync` | Python/Shell (ghcr.io/helloworld44-89/nebula-sync:latest) | Sync daemon |
| `nebula-sync-secrets` | Kubernetes Secret | API keys for Pi-hole instances |

## Data Flow

```
Cron (every hour)
    │
    ▼
Primary Pi-hole (192.168.5.5)
    │  API: /api/custom.list
    ▼
Nebula Sync Pod
    │  Diff + Transform
    ▼
Secondary Pi-hole (192.168.5.6)
    │  API: /api/custom.list (POST)
    ▼
Pi-hole FTL reload
```

## Network

- **Outbound**: HTTPS to both Pi-hole instances (ports 443)
- **No Ingress**: No external exposure
- **Internal**: Cluster DNS for service discovery

## Storage

- **None**: Fully stateless
- Config/Secrets via k8s Secret `nebula-sync-secrets`

## Secrets

```yaml
# nebula-sync-secrets (namespace: tools)
PRIMARY_PIHOLE_URL: "https://192.168.5.5"
PRIMARY_PIHOLE_API_KEY: "<api-key>"
SECONDARY_PIHOLE_URL: "https://192.168.5.6"
SECONDARY_PIHOLE_API_KEY: "<api-key>"
```

## Schedule

- **Cron**: `0 * * * *` (hourly at minute 0)
- **Manual trigger**: `kubectl create job --from=cronjob/nebula-sync manual-sync-$(date +%s)`

## Resources

```yaml
requests:
  cpu: 10m
  memory: 32Mi
limits:
  cpu: 100m
  memory: 128Mi
```

## Health Checks

- **Liveness**: HTTP GET `/healthz` (if implemented)
- **Startup**: Immediate success
- **Success metric**: Job completes with exit code 0

## Logs

- Stdout: Sync summary (records added/removed/unchanged)
- Stderr: Errors (API failures, auth issues, network)
- View: `kubectl logs -n tools -l app.kubernetes.io/instance=nebula-sync`