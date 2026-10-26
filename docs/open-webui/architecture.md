# Open WebUI Architecture

## Overview

Open WebUI is a self-hosted AI chat interface supporting multiple LLM backends (Ollama, OpenAI-compatible APIs). Features RAG, multi-model conversations, and a modern React frontend.

## Components

| Component | Technology | Purpose |
|---|---|---|
| `open-webui` | Python/FastAPI + React (ghcr.io/open-webui/open-webui:latest) | Web server + frontend |
| `open-webui-data` | File-based (local-path PVC) | SQLite DB, vector DB, uploads, cache |

## Data Flow

```
Browser
    │
    ▼ HTTPS
Traefik Ingress
    │
    ▼ HTTP
Open WebUI (port 8080)
    │
    ├──▶ SQLite DB (/app/backend/data/webui.db)
    ├──▶ Vector DB (/app/backend/data/vector_db/)
    ├──▶ Uploads (/app/backend/data/uploads/)
    └──▶ Cache (/app/backend/data/cache/)
    │
    ▼ LLM API calls
External LLM (Ollama / OpenAI API)
```

## Network

- **Ingress**: `ai.conderhome.xyz` → Traefik → Open WebUI service (port 8080)
- **TLS**: Wildcard cert `wildcard-conderhome-xyz-tls`
- **Internal**: HTTP on port 8080
- **LLM Backend**: Outbound to Ollama (if local) or OpenAI API

## Storage

- **PVC**: `open-webui-data` (local-path, RWO)
- **Node**: Pinned to `prod-k3s-1` (192.168.5.60)
- **Contents**: `webui.db` (SQLite), `vector_db/` (chroma), `uploads/`, `cache/`

## Security

- **Auth**: Built-in user management (email/password, OAuth)
- **Admin**: First user becomes admin
- **API Keys**: Per-user API keys for programmatic access
- **RAG**: Document uploads stored in uploads/, vectorized in vector_db/
- **No encryption at rest** (relies on PVC/node security)

## Resources

```yaml
requests:
  cpu: 100m
  memory: 512Mi
limits:
  cpu: 2000m
  memory: 4Gi
```

## Health Checks

- **Liveness/Readiness**: `GET /health` on port 8080
- Initial delay: 30s (model loading)
- Period: 30s

## LLM Backends

- **Ollama**: `http://ollama:11434` (if deployed in-cluster)
- **OpenAI API**: `https://api.openai.com/v1`
- **OpenAI-compatible**: Any OpenAI-compatible endpoint
- Configured via UI: Settings → Models