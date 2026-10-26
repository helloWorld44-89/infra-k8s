# wg-easy Architecture

## Overview

wg-easy is a Web UI for managing WireGuard VPN configurations. Runs as a privileged container with host networking to manage kernel WireGuard interfaces.

## Components

| Component | Technology | Purpose |
|---|---|---|
| `wg-easy` | Node.js/Go (ghcr.io/wg-easy/wg-easy:latest) | Web UI + WireGuard manager |
| `wg-easy-config` | HostPath / PVC | WireGuard configs, private keys, peer configs |

## Data Flow

```
Browser
    │
    ▼ HTTPS
Traefik Ingress
    │
    ▼ HTTP
wg-easy (port 51821)
    │
    ├──▶ WireGuard kernel module (host)
    │    ├── wg0 interface
    │    │   ├── Private key
    │    │   │   └── Peers (public keys, allowed IPs, endpoints)
    │    └── wg-quick / wg command
    │
    └──▶ Config volume (/etc/wireguard)
         ├── wg0.conf
         └── peers/ (per-peer configs)
```

## Network

- **Host Network**: `hostNetwork: true` (required for WireGuard kernel access)
- **Pinned Node**: `prod-k3s-master` (192.168.5.65)
- **Ports**:
  - `51820/udp` - WireGuard traffic (host)
  - `51821/tcp` - Web UI (host)
- **Ingress**: `wg-easy.conderhome.xyz` → Traefik → wg-easy service (port 51821)
- **TLS**: Wildcard cert `wildcard-conderhome-xyz-tls`
- **Router Port Forward**: UDP 51820 → 192.168.5.65

## Storage

| Volume | Type | Path | Contents |
|---|---|---|---|
| wg-easy-config | HostPath / PVC | /etc/wireguard | `wg0.conf`, `privatekey`, `peers/*.conf` |

## Security

- **Capabilities**: `NET_ADMIN`, `SYS_MODULE` (required for WireGuard)
- **Host Network**: Full host network access
- **Privileged**: Effectively yes (via capabilities)
- **Auth**: Built-in password auth (bcrypt)
- **WireGuard**: Private key never leaves server; peers get public key only
- **Firewall**: Only UDP 51820 + TCP 51821 exposed via router

## Capabilities

```yaml
capabilities:
  add:
    - NET_ADMIN
    - SYS_MODULE
```

## Resources

```yaml
requests:
  cpu: 50m
  memory: 64Mi
limits:
  cpu: 500m
  memory: 256Mi
```

## Health Checks

- **Liveness/Readiness**: `GET /` on port 51821
- Initial delay: 10s
- Period: 30s

## Key Management

- **Server Private Key**: Generated on first run, stored in `/etc/wireguard/privatekey`
- **Server Public Key**: Derived, shared with peers
- **Peer Keys**: Generated per peer, stored in `/etc/wireguard/peers/`
- **Rotation**: Manual (regenerate keys, redistribute peer configs)

## Migration

- **Keys**: Must preserve exact private key for zero-client-reconfig migration
- **Verification**: `wg pubkey < privatekey` must match original public key
- **Zero Reconfig**: Peers unchanged if server public key identical