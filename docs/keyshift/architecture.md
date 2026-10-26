# KeyShift Architecture

## Overview

KeyShift is a Flask-based WiFi network manager for managing SSID/password rotations and QR code generation for guest networks. Deployed as a simple web UI.

## Components

| Component | Technology | Purpose |
|---|---|---|
| `keyshift` | Python/Flask (ghcr.io/helloworld44-89/keyshift:latest) | Web application |
| `keyshift-config` | JSON file (local-path PVC) | Configuration (SSIDs, passwords, schedules) |
| `keyshift-db` | SQLite (local-path PVC) | Audit logs, rotation history |

## Data Flow

```
Browser
    │
    ▼ HTTPS (TLS at Traefik)
Traefik Ingress
    │
    ▼ HTTP
KeyShift Pod (port 5000)
    │
    ├──▶ Config PVC (/app/app/config/file)
    └──▶ DB PVC (/app/instance/KeyShift.db)
```

## Network

- **Ingress**: `keyshift.conderhome.xyz` → Traefik → KeyShift service (port 5000)
- **TLS**: Wildcard cert `wildcard-conderhome-xyz-tls`
- **Internal**: HTTP on port 5000

## Storage

| PVC | Size | Mount Path | Contents |
|---|---|---|---|
| keyshift-config | 1Gi | /app/app/config/file | `config.json` |
| keyshift-db | 1Gi | /app/instance | `KeyShift.db` (SQLite) |

## Security

- **Authentication**: Flask-Login session cookies (Secure, HttpOnly, SameSite=Lax)
- **Rate limiting**: Flask-Limiter (20 req/hr on `/` - probes use `/healthz`)
- **CSRF**: Flask-WTF CSRF protection
- **Secrets**: `SECRET_KEY` from k8s Secret `keyshift-secrets`

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

- **Liveness/Readiness**: `GET /healthz` (exempt from rate limiting)
- Initial delay: 15s / 5s
- Period: 20s / 10s

## Known Issues

- **Flask-Limiter**: Rate limits `/` at 20 req/hr - kubelet probes must use `/healthz`
- **Gunicorn workers**: 4 workers at 256Mi limit may OOM - monitor memory