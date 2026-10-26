# wg-easy API Documentation

## Base URL

```
https://wg-easy.conderhome.xyz
```

## Authentication

Session-based (cookie) or HTTP Basic Auth for API.

### Web UI

1. `GET /` → Login page
2. `POST /api/session` → `{password: "..."}` → Sets session cookie
3. Subsequent requests: Cookie auto-sent

### API (with cookie)

All endpoints require valid session cookie.

## Endpoints

### WireGuard Interface

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/wg0` | Get interface status |
| `PUT` | `/api/wg0` | Update interface config |

### Peers

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/wg0/peers` | List all peers |
| `POST` | `/api/wg0/peers` | Create peer |
| `GET` | `/api/wg0/peers/{id}` | Get peer details |
| `PUT` | `/api/wg0/peers/{id}` | Update peer |
| `DELETE` | `/api/wg0/peers/{id}` | Delete peer |
| `GET` | `/api/wg0/peers/{id}/qrcode` | QR code for peer config |
| `GET` | `/api/wg0/peers/{id}/config` | Download .conf file |

### Peer Object

```json
{
  "id": "uuid",
  "name": "phone-justin",
  "publicKey": "base64...",
  "presharedKey": "base64...",
  "allowedIPs": ["10.0.0.2/32"],
  "endpoint": "auto",
  "persistentKeepalive": 25,
  "dns": "1.1.1.1",
  "mtu": 1420,
  "createdAt": "2026-10-25T...",
  "lastHandshake": "2026-10-25T...",
  "transferRx": 1024000,
  "transferTx": 512000
}
```

### Server Config

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/wg0/config` | Get server config |
| `PUT` | `/api/wg0/config` | Update server config |

Server config fields:
- `interface`: "wg0"
- `address`: ["10.0.0.1/24"]
- `port`: 51820
- `dns`: "1.1.1.1"
- `mtu`: 1420
- `preUp`: ""
- `postUp`: ""
- `preDown`: ""
- `postDown`: ""

### Stats

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/wg0/stats` | Real-time transfer stats |

## Authentication

### Session

```
POST /api/session
Content-Type: application/json

{"password": "your-admin-password"}
```

Response: Sets `session` cookie.

### Logout

```
POST /api/session/logout
```

## WebSocket

Real-time stats:
- `wss://wg-easy.conderhome.xyz/api/wg0/ws`
- Streams peer transfer updates

## Health Check

```
GET /
```
Returns HTML (200)

## Rate Limits

- Login: 5 attempts/min per IP
- API: 100 req/min per session