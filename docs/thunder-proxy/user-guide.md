# Thunder Proxy User Guide

## Access

Navigate to `https://thunder.conderhome.xyz`

Proxmox login page loads. Authenticate with Proxmox credentials.

## What It Proxies

Full Proxmox VE Web UI and API:
- Web GUI: Dashboard, VMs, Containers, Storage, Network, Datacenter
- API: All Proxmox REST endpoints (for automation/scripts)

## Use Cases

- Access Proxmox UI via friendly hostname (`thunder.conderhome.xyz`)
- Same wildcard TLS cert as other services
- API automation via `thunder.conderhome.xyz/api2/...`
- Single ingress point for all homelab services

## Proxmox Features Available

All standard Proxmox features work through the proxy:
- VM/CT lifecycle (start/stop/snapshot/migrate)
- Storage management
- Network/SDN configuration
- Backup/Restore
- HA configuration
- User/permission management

## Authentication

Use Proxmox credentials directly:
- Username: `root@pam` (or other realm)
- Password: Proxmox password
- 2FA: If configured on Proxmox

## API Access

```bash
# Get ticket
curl -k -X POST https://thunder.conderhome.xyz/api2/json/access/ticket \
  -d "username=root@pam" -d "password=<password>"

# Use ticket for subsequent requests
curl -k -H "CSRFPreventionToken: <ticket>" \
  -b "PVEAuthCookie=<ticket>" \
  https://thunder.conderhome.xyz/api2/json/nodes

# Or use API token (recommended for automation)
curl -k -H "Authorization: PVEAPIToken=root@pam!tokenid=secret" \
  https://thunder.conderhome.xyz/api2/json/cluster/resources
```

## Troubleshooting

| Issue | Fix |
|---|---|
| 502 Bad Gateway | Traefik can't reach 192.168.5.10:8006 - check network |
| Cert warning | Proxmox uses self-signed cert - backend trust via ServersTransport |
| Login fails | Check Proxmox directly at 192.168.5.10:8006 |
| API 401 | Verify ticket/token, check CSRF header for ticket auth |