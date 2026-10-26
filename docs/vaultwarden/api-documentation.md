# Vaultwarden API Documentation

## Base URL

```
https://vault.conderhome.xyz
```

## Authentication

Vaultwarden implements the full Bitwarden API. Authentication follows the Bitwarden specification:

1. **Identity endpoint**: `POST /identity/connect/token`
   - `grant_type=password`
   - `username=<email>`
   - `password=<master_password>`
   - `scope=api offline_access`
   - `client_id=<client_id>`
   - `client_secret=<client_secret>` (optional for public clients)
   - `device=<device_identifier>`

2. **Access token**: Bearer token in `Authorization` header

## Key Endpoints

### Vault Operations

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/v1/vault` | List all vault items |
| `POST` | `/api/v1/vault` | Create vault item |
| `GET` | `/api/v1/vault/{id}` | Get vault item |
| `PUT` | `/api/v1/vault/{id}` | Update vault item |
| `DELETE` | `/api/v1/vault/{id}` | Delete vault item |

### Folders

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/v1/folders` | List folders |
| `POST` | `/api/v1/folders` | Create folder |
| `PUT` | `/api/v1/folders/{id}` | Update folder |
| `DELETE` | `/api/v1/folders/{id}` | Delete folder |

### Collections (Organizations)

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/v1/collections` | List collections |
| `POST` | `/api/v1/collections` | Create collection |
| `PUT` | `/api/v1/collections/{id}` | Update collection |
| `DELETE` | `/api/v1/collections/{id}` | Delete collection |

### Sends

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/v1/sends` | List sends |
| `POST` | `/api/v1/sends` | Create send |
| `GET` | `/api/v1/sends/{id}` | Get send |
| `PUT` | `/api/v1/sends/{id}` | Update send |
| `DELETE` | `/api/v1/sends/{id}` | Delete send |

### Attachments

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/v1/attachments` | Upload attachment |
| `GET` | `/api/v1/attachments/{id}` | Download attachment |
| `DELETE` | `/api/v1/attachments/{id}` | Delete attachment |

### User/Account

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/v1/accounts/verify` | Verify email |
| `POST` | `/api/v1/accounts/2fa/email` | Request 2FA email |
| `GET` | `/api/v1/public-key` | Get RSA public key |

## Rate Limits

- No explicit rate limiting configured at application level
- Traefik ingress provides basic DDoS protection
- Consider adding middleware for production use

## Client Libraries

Official Bitwarden SDKs work directly:
- **JavaScript/TypeScript**: `@bitwarden/sdk`
- **Python**: `bitwarden-sdk`
- **Go**: `github.com/bitwarden/sdk-go`
- **CLI**: `bw` (official Bitwarden CLI)

## WebSocket

Real-time sync via WebSocket:
- Endpoint: `wss://vault.conderhome.xyz/notifications/hub`
- Requires valid access token
- Pushes vault changes, sync events