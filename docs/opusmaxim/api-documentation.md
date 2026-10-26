# OpusMaxim API Documentation

## Base URL

```
https://opusmaxim.conderhome.xyz
```

## Authentication

Session-based via Flask-Login.

### Endpoints

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/` | Homepage / portfolio |
| `GET` | `/login` | Login page |
| `POST` | `/login` | Authenticate |
| `GET` | `/logout` | Destroy session |
| `GET` | `/register` | Register (if enabled) |
| `POST` | `/register` | Create account |

### Project/Portfolio

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/projects` | List projects |
| `GET` | `/projects/{id}` | Project detail |
| `GET` | `/api/projects` | JSON list (if implemented) |

### Avatar Upload

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/upload/avatar` | Upload profile avatar |
| `GET` | `/avatars/{filename}` | Serve avatar |

## Health Check

```
GET /
```
Returns HTML (200 OK)

## Authentication Errors

- 401: Login required
- 403: Insufficient permissions
- 429: Rate limited (if configured)