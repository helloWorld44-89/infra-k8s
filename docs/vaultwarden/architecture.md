# Vaultwarden Architecture

## Overview

Vaultwarden is a lightweight, self-hosted Bitwarden-compatible password manager written in Rust. It provides full Vaultwarden API compatibility for clients (web vault, browser extensions, mobile apps, CLI).

## Components

| Component | Technology | Purpose |
|---|---|---|
| `vaultwarden` | Rust (vaultwarden/server:1.37.4) | Main application server |
| `vaultwarden-data` | SQLite (local-path PVC) | Encrypted vault storage, attachments, sends, RSA keys |

## Data Flow

```
Client (Browser/Mobile/CLI)
    │
    ▼ HTTPS (TLS termination at Traefik)
Traefik Ingress (wildcard *.conderhome.xyz cert)
    │
    ▼ HTTP (internal)
Vaultwarden Pod (port 80)
    │
    ▼ SQLite
PVC (vaultwarden-data) → local-path storage on prod-k3s-3
```

## Network

- **Ingress**: `vault.conderhome.xyz` → Traefik → Vaultwarden service (port 80)
- **TLS**: Terminated at Traefik with wildcard cert `wildcard-conderhome-xyz-tls`
- **Internal**: Plain HTTP between Traefik and Vaultwarden pod

## Storage

- **PVC**: `vaultwarden-data` (1Gi, local-path, RWO)
- **Node**: Pinned to `prod-k3s-3` (192.168.5.62)
- **Contents**: `db.sqlite3` + WAL/SHM files, `attachments/`, `sends/`, `rsa_keys/`

## Security

- **Encryption**: All vault data encrypted client-side before reaching server
- **Admin token**: Not configured (admin panel disabled)
- **Rate limiting**: Not configured at app level (relies on Traefik/TLS)
- **2FA**: Supported via TOTP, Duo, YubiKey, Email

## Resources

```yaml
requests:
  cpu: 20m
  memory: 64Mi
limits:
  cpu: 500m
  memory: 256Mi
```

## Health Checks

- **Liveness**: `GET /alive` every 20s (initialDelay 15s)
- **Readiness**: `GET /alive` every 10s (initialDelay 5s)

## Backup Strategy

- SQLite database backed up via weekly cron (manual process)
- Attachments and sends stored in same PVC
- Consider periodic PVC snapshot or `sqlite3 .backup` to NFS