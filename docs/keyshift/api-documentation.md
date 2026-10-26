# KeyShift API Documentation

## Base URL

```
https://keyshift.conderhome.xyz
```

## Authentication

Session-based authentication via Flask-Login.

1. `POST /login` - username/password → sets session cookie
2. Subsequent requests: session cookie automatically sent
3. `GET /logout` - destroys session

## Endpoints

### Configuration

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/config` | Get current config |
| `POST` | `/api/config` | Update config (admin) |

### SSID Management

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/ssids` | List all SSIDs |
| `POST` | `/api/ssids` | Create SSID |
| `PUT` | `/api/ssids/{id}` | Update SSID |
| `DELETE` | `/api/ssids/{id}` | Delete SSID |

### Rotation / Scheduling

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/schedule` | Get rotation schedule |
| `POST` | `/api/schedule` | Create schedule entry |
| `PUT` | `/api/schedule/{id}` | Update schedule |
| `DELETE` | `/api/schedule/{id}` | Delete schedule |

### QR Code Generation

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/qr/{ssid_id}` | Generate QR for SSID (guest config) |

### Audit / Logs

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/log` | Get rotation audit logs |

## Health Check

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/healthz` | Health check (rate-limit exempt) |

Returns: `{"status": "ok"}`

## Rate Limiting

- **Root `/`**: 20 requests/hour per IP (Flask-Limiter)
- **`/healthz`**: Exempt (`@limiter.exempt`)
- **API endpoints**: Not explicitly limited (add if needed)

## Rate Limit Headers

Responses include:
- `X-RateLimit-Limit`: Max requests
- `X-RateLimit-Remaining`: Remaining in window
- `X-RateLimit-Reset`: Unix timestamp when window resets

429 response includes `Retry-After` header.

## Authentication Errors

- 401: Invalid credentials / session expired
- 403: Insufficient permissions (admin required)
- 429: Rate limited - check `Retry-After`