# wg-easy User Guide

## Access

Navigate to `https://wg-easy.conderhome.xyz`

Enter admin password (stored in k8s secret).

## Dashboard

- **Interface status**: Up/Down, listening port
- **Transfer stats**: Total Rx/Tx
- **Peer list**: Name, last handshake, transfer, actions

## Managing Peers

### Adding a Peer

1. Click "New Client"
2. Enter name (e.g., "justin-iphone")
3. Optional: Set allowed IPs (default: next available /32)
3. Optional: Set DNS (default: 1.1.1.1)
4. Click "Create"

### Peer Actions

- **QR Code**: Scan with WireGuard mobile app
- **Download .conf**: For desktop WireGuard clients
- **Enable/Disable**: Toggle without deleting
- **Edit**: Change name, allowed IPs, DNS, keepalive
- **Delete**: Remove permanently (cannot undo)

### Peer Config

Each peer gets:
- Private key (generated)
- Preshared key (optional, extra security)
- Allowed IPs (e.g., `10.0.0.2/32`)
- DNS (default: 1.1.1.1)
- MTU (default: 1420)
- Keepalive: 25s (for NAT traversal)

## Server Config

- **Interface**: wg0
- **Address**: `10.0.0.1/24` (server IP in VPN)
- **Port**: 51820 (UDP)
- **DNS**: 1.1.1.1 (pushed to peers)
- **MTU**: 1420

## Mobile Setup

### iOS / Android

1. Install WireGuard app
2. In wg-easy: Click peer → QR Code
3. Scan QR in WireGuard app
4. Name tunnel, enable
4. Verify: "Handshake" appears, data transfers

### Desktop (Linux/macOS/Windows)

1. Download .conf from peer actions
2. Import in WireGuard client
3. Activate tunnel

## Router Setup

**Port Forward Required**:
- UDP 51820 → 192.168.5.65 (k3s master)
- TCP 51821 → 192.168.5.65 (Web UI, optional if using Traefik)

**Firewall**: Allow UDP 51820 inbound

## Troubleshooting

| Issue | Fix |
|---|---|
| No handshake | Check UDP 51820 port forward, firewall |
| Connected but no internet | Check AllowedIPs, server PostUp/PostDown rules |
| Slow speeds | Check MTU (try 1280), check CPU on server |
| QR code invalid | Regenerate peer, scan fresh |
| Mobile disconnects | Enable PersistentKeepalive (25s) |

## Backup

- Config: `/etc/wireguard/wg0.conf` + peers
- Keys: Never expose private keys in backups
- Peer configs: Export individually via UI