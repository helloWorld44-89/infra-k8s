# Actual Budget API Documentation

## Base URL

```
https://budget.conderhome.xyz
```

## Authentication

Session-based via Actual's built-in auth. No public REST API for budget data (client-side encryption). API primarily used for sync protocol.

## Sync Protocol

Actual uses a custom file-based sync protocol over HTTP:

1. **Manifest**: `GET /sync/manifest` - List files + versions
2. **Download**: `GET /sync/files/{filename}` - Get file content
3. **Upload**: `POST /sync/files/{filename}` - Push changes
4. **Delete**: `DELETE /sync/files/{filename}` - Remove file

Files are encrypted client-side; server stores opaque blobs.

## User Endpoints

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/auth/login` | Login (email/password) |
| `POST` | `/api/auth/register` | Register account |
| `POST` | `/api/auth/logout` | Logout |
| `GET` | `/api/auth/me` | Current user info |

## Budget Files

- Each budget = encrypted SQLite database (`.actual` file)
- Stored in `/data/user-files/{user-id}/`
- Sync metadata in `/data/user-files/{user-id}/.sync/`

## Server Files

- Migrations: `/data/server-files/migrations/`
- Static assets served from `/data/server-files/public/`

## Health Check

| Method | Endpoint |
|---|---|
| `GET` | `/` |

Returns React app HTML (200 OK).

## Rate Limits

- No explicit rate limiting
- Sync protocol self-throttles via client

## Authentication Errors

- 401: Invalid credentials / session expired
- 403: Access denied (wrong user's files)