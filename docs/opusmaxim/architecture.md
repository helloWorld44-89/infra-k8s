# OpusMaxim Architecture

## Overview

OpusMaxim is a Flask-based personal portfolio / project showcase application with SQLite backend. Serves a React frontend via Flask templates.

## Components

| Component | Technology | Purpose |
|---|---|---|
| `opusmaxim` | Python/Flask (ghcr.io/helloworld44-89/opusmaxim:latest) | Web application |
| `opusmaxim-data` | SQLite (local-path PVC) | Database + avatar uploads |

## Data Flow

```
Browser
    │
    ▼ HTTPS
Traefik Ingress
    │
    ▼ HTTP
OpusMaxim (port 5000)
    │
    ├──▶ SQLite (/app/instance/opus.db)
    └──▶ Avatars (/app/instance/avatars/)
```

## Network

- **Ingress**: `opusmaxim.conderhome.xyz` → Traefik → OpusMaxim service (port 5000)
- **TLS**: Wildcard cert `wildcard-conderhome-xyz-tls`
- **Internal**: HTTP on port 5000

## Storage

| PVC | Size | Mount | Contents |
|---|---|---|---|
| opusmaxim-data | 1Gi | /app/instance | `opus.db` (SQLite), `avatars/` |

## Security

- **Auth**: Flask-Login sessions (Secure, HttpOnly, SameSite=Lax)
- **CSRF**: Flask-WTF
- **Passwords**: bcrypt
- **Secrets**: `SECRET_KEY` from k8s Secret `opusmaxim-secrets`

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

- **Liveness/Readiness**: `GET /` on port 5000
- Initial delay: 15s / 5s
- Period: 20s / 10s