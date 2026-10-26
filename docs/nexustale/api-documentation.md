# NexusTale API Documentation

## Base URL

```
https://nexustale.conderhome.xyz/api
```

## Authentication

JWT-based (access + refresh tokens).

### Login

```
POST /auth/login
Content-Type: application/x-www-form-urlencoded

username=<email>&password=<password>
```

Response:
```json
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ...",
  "token_type": "bearer"
}
```

### Usage

```bash
Authorization: Bearer <access_token>
```

### Token Refresh

```
POST /auth/refresh
Authorization: Bearer <refresh_token>
```

## Endpoints

### Notes

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/notes` | List notes (paginated, filterable) |
| `POST` | `/notes` | Create note |
| `GET` | `/notes/{id}` | Get note |
| `PUT` | `/notes/{id}` | Update note |
| `DELETE` | `/notes/{id}` | Delete note |
| `POST` | `/notes/{id}/embed` | Generate embedding |

### Collections

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/collections` | List collections |
| `POST` | `/collections` | Create collection |
| `PUT` | `/collections/{id}` | Update collection |
| `DELETE` | `/collections/{id}` | Delete collection |

### Tags

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/tags` | List tags |
| `POST` | `/tags` | Create tag |

### Search / RAG

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/search` | Semantic search (vector) |
| `POST` | `/search/hybrid` | Hybrid (keyword + vector) |
| `GET` | `/search/suggestions` | Autocomplete |

### Files / Attachments

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/files` | Upload file |
| `GET` | `/files/{id}` | Download |
| `DELETE` | `/files/{id}` | Delete |

### User / Settings

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/users/me` | Current user |
| `PUT` | `/users/me` | Update profile |
| `PUT` | `/users/me/password` | Change password |
| `GET` | `/settings` | Get settings |
| `PUT` | `/settings` | Update settings |

## Health

```
GET /health
```
Returns: `{"status": "ok", "database": "connected", "redis": "connected"}`

## Rate Limiting

- Auth endpoints: 10 req/min per IP
- API endpoints: 100 req/min per user
- Search: 30 req/min per user

Headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `Retry-After`