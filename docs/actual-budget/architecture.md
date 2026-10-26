# Actual Budget Architecture

## Overview

Actual Budget is a local-first, privacy-focused personal finance app built with Node.js/React. Self-hosted version runs the Actual server with file-based sync.

## Components

| Component | Technology | Purpose |
|---|---|---|
| `actual-budget` | Node.js/React (actualbudget/actual-server:latest) | Web server + API |
| `actual-budget-data` | File-based (local-path PVC) | User files, budgets, sync data |

## Data Flow

```
Browser
    │
    ▼ HTTPS
Traefik Ingress
    │
    ▼ HTTP
Actual Server (port 5006)
    │
    ▼ File I/O
PVC (actual-budget-data) → local-path on prod-k3s-3
    ├── /data/user-files/ (budgets, sync)
    ├── /data/server-files/ (migrations)
    └── /data/.migrate (migration marker)
```

## Network

- **Ingress**: `budget.conderhome.xyz` → Traefik → Actual service (port 5006)
- **TLS**: Wildcard cert `wildcard-conderhome-xyz-tls`
- **Internal**: HTTP on port 5006

## Storage

- **PVC**: `actual-budget-data` (1Gi, local-path, RWO)
- **Node**: Pinned to `prod-k3s-3` (192.168.5.62)
- **Structure**: `/data/user-files/`, `/data/server-files/`, `.migrate`

## Security

- **Encryption**: End-to-end encryption client-side (user password derives key)
- **Sync**: File-based sync protocol (no server-side decryption)
- **Auth**: Built-in user accounts with bcrypt password hashing
- **Server-side**: No access to decrypted data

## Resources

```yaml
requests:
  cpu: 20m
  memory: 128Mi
limits:
  cpu: 500m
  memory: 512Mi
```

## Health Checks

- **Liveness/Readiness**: `GET /` on port 5006
- Initial delay: 15s / 5s
- Period: 20s / 10s

## Migration

- Auto-runs migrations on startup (`Migrations: DONE` in logs)
- SQLite database per budget file
- Migration marker: `.migrate` file