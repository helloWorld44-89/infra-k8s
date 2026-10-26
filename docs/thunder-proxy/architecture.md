# Thunder Proxy Architecture

## Overview

Thunder Proxy is a Traefik IngressRoute that routes `thunder.conderhome.xyz` to the Proxmox web UI on Balerion (192.168.5.10:8006). Demonstrates routing external (non-k3s) services through k3s's Traefik ingress.

## Components

| Component | Technology | Purpose |
|---|---|---|
| `thunder-proxmox` | ExternalName Service | Points Traefik to 192.168.5.10 |
| `thunder-proxmox-transport` | ServersTransport | Trusts Proxmox self-signed cert on backend |
| `thunder-proxmox` | IngressRoute | Routes thunder.conderhome.xyz to Proxmox |

## Data Flow

```
Browser
    │
    ▼ HTTPS (TLS: wildcard cert)
Traefik Ingress (k3s)
    │
    ▼ HTTPS (backend, self-signed cert trusted via ServersTransport)
Proxmox Web UI (192.168.5.10:8006)
```

## Network

- **Frontend**: `thunder.conderhome.xyz` → Traefik (port 443, wildcard TLS)
- **Backend**: Traefik → 192.168.5.10:8006 (HTTPS, Proxmox self-signed cert)
- **ServersTransport**: `insecureSkipVerify: true` (trusts Proxmox self-signed cert)
- **No Ingress on Proxmox**: Proxmox remains on its native port 8006

## Security

- **Frontend TLS**: Real wildcard cert (`wildcard-conderhome-xyz-tls`)
- **Backend TLS**: Proxmox self-signed cert (trusted via `insecureSkipVerify`)
- **No credentials in k8s**: Auth handled by Proxmox directly
- **Network**: Traefik → Proxmox over LAN (192.168.5.0/24)

## Resources

- No pods, no PVCs, no resources
- Pure Traefik CRDs (Service, ServersTransport, IngressRoute)

## Requirements

- **Traefik Feature**: `allowExternalNameServices=true` (set via HelmChartConfig)
- **DNS**: `thunder.conderhome.xyz` → 192.168.5.65 (Pi-hole)
- **Network**: Traefik pod can reach 192.168.5.10:8006