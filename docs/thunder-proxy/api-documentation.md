# Thunder Proxy API Documentation

## Base URL

```
https://thunder.conderhome.xyz
```

## Proxied Service

Proxmox VE Web API (passed through transparently).

## Proxmox API Endpoints (Selected)

### Authentication

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api2/json/access/ticket` | Get auth ticket |
| `POST` | `/api2/json/access/tfa` | 2FA challenge |

### Nodes

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api2/json/nodes` | List nodes |
| `GET` | `/api2/json/nodes/{node}` | Node status |
| `GET` | `/api2/json/nodes/{node}/qemu` | List VMs |
| `GET` | `/api2/json/nodes/{node}/lxc` | List containers |
| `GET` | `/api2/json/nodes/{node}/storage` | List storage |
| `GET` | `/api2/json/nodes/{node}/network` | Network config |

### VMs

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api2/json/nodes/{node}/qemu/{vmid}/status/current` | VM status |
| `POST` | `/api2/json/nodes/{node}/qemu/{vmid}/status/start` | Start VM |
| `POST` | `/api2/json/nodes/{node}/qemu/{vmid}/status/stop` | Stop VM |
| `POST` | `/api2/json/nodes/{node}/qemu/{vmid}/status/shutdown` | Shutdown VM |
| `POST` | `/api2/json/nodes/{node}/qemu/{vmid}/status/reboot` | Reboot VM |
| `GET` | `/api2/json/nodes/{node}/qemu/{vmid}/config` | VM config |
| `PUT` | `/api2/json/nodes/{node}/qemu/{vmid}/config` | Update VM config |

### Cluster

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api2/json/cluster/resources` | All resources (VMs, CTs, storage) |
| `GET` | `/api2/json/cluster/status` | Cluster status |

### API Token Auth

```bash
# Header
Authorization: PVEAPIToken=<user>@<realm>!<tokenid>=<secret>

# Example
Authorization: PVEAPIToken=root@pam!hermes=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

## Thunder Proxy Specific

No additional endpoints. All requests proxied transparently to Proxmox.

## Health Check

```
GET /
```
Returns Proxmox login page (200) or Proxmox API response.