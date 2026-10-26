# Open WebUI API Documentation

## Base URL

```
https://ai.conderhome.xyz
```

## Authentication

### Session (Web UI)

1. `POST /api/v1/auths/signin` - email/password → JWT in cookie
2. `POST /api/v1/auths/signup` - register (first user = admin)
3. `POST /api/v1/auths/signout` - destroy session

### API Keys (Programmatic)

1. UI: Settings → API Keys → Generate
2. Header: `Authorization: Bearer <api-key>`
3. Scoped to user permissions

## Endpoints

### Chat / Completions

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/chat/completions` | OpenAI-compatible chat completion |
| `POST` | `/api/chat/completions/stream` | Streaming response |

Request format (OpenAI-compatible):
```json
{
  "model": "llama3.2",
  "messages": [{"role": "user", "content": "Hello"}],
  "stream": false,
  "temperature": 0.7
}
```

### Models

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/models` | List available models |
| `POST` | `/api/models/pull` | Pull model from Ollama |
| `DELETE` | `/api/models/{id}` | Delete local model |

### Knowledge / RAG

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/knowledge` | List knowledge bases |
| `POST` | `/api/knowledge` | Create knowledge base |
| `POST` | `/api/knowledge/{id}/files` | Upload document |
| `POST` | `/api/knowledge/{id}/query` | Query RAG |

### Files / Uploads

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/files` | Upload file (for RAG/chat) |
| `GET` | `/api/files/{id}` | Get file info |
| `DELETE` | `/api/files/{id}` | Delete file |

### Users / Admin

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/users` | List users (admin) |
| `POST` | `/api/users` | Create user (admin) |
| `PUT` | `/api/users/{id}` | Update user (admin) |
| `DELETE` | `/api/users/{id}` | Delete user (admin) |
| `GET` | `/api/admin/settings` | Get global settings (admin) |
| `PUT` | `/api/admin/settings` | Update settings (admin) |

### Prompts / Functions

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/prompts` | List prompt templates |
| `POST` | `/api/prompts` | Create prompt |
| `GET` | `/api/functions` | List functions/tools |
| `POST` | `/api/functions` | Register function |

## WebSocket

Real-time streaming:
- Endpoint: `wss://ai.conderhome.xyz/api/chat/ws`
- Auth: JWT token in query param
- Streams token-by-token for chat completions

## Health Check

| Method | Endpoint |
|---|---|
| `GET` | `/health` |

Returns: `{"status": "ok"}`

## Rate Limits

- No explicit rate limiting
- LLM backend may have its own limits