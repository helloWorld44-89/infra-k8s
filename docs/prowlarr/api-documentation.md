---
title: Prowlarr API Documentation
date: 2026-10-23
author: Hermes Docs Agent
tags: [api, documentation, media, indexers, swagger]
---

# Prowlarr API Documentation

## Overview

Prowlarr exposes a REST API (the standard *arr-family `/api/v1` surface) for managing indexers, applications (Radarr/Sonarr sync targets), indexer proxies (including FlareSolverr), and system status. The same API backs Prowlarr's own web UI, and is what Radarr/Sonarr use on the other side of the sync relationship.

**Base URL (in-cluster)**: `http://prowlarr.media.svc.cluster.local:9696/api/v1`

**Base URL (external)**: `https://prowlarr.conderhome.xyz/api/v1`

**Authentication**: API key, passed via the `X-Api-Key` header. The key is generated on first run and visible/regenerable under **Settings → General → Security** in the Prowlarr UI.

**Content-Type**: `application/json`

## Authentication

### Retrieving the API Key

The API key lives in Prowlarr's config database (`/config/config.xml` inside the pod) and in the UI under **Settings → General**. It is not stored as a Kubernetes Secret in this deployment.

```bash
# Read it directly from the running pod's config (for ad-hoc scripting)
kubectl exec -n media deploy/prowlarr -- cat /config/config.xml | grep -i apikey
```

### Using the API Key

```bash
curl -H "X-Api-Key: ***" \
     -H "Content-Type: application/json" \
     "https://prowlarr.conderhome.xyz/api/v1/indexer"
```

## API Endpoints

### Indexers API

#### List Indexers

```http
GET /api/v1/indexer
```

Returns all configured indexers.

**Response:**
```json
[
  {
    "id": 3,
    "name": "ExampleTracker",
    "enable": true,
    "protocol": "torrent",
    "privacy": "private",
    "indexerUrls": ["https://example-tracker.example"],
    "tags": [],
    "fields": [
      { "name": "apiKey", "value": "***" }
    ],
    "indexerProxy": null,
    "capabilities": {
      "categories": [
        { "id": 2000, "name": "Movies" },
        { "id": 5000, "name": "TV" }
      ]
    },
    "priority": 25,
    "added": "2026-09-01T12:00:00Z"
  }
]
```

#### Get Indexer Details

```http
GET /api/v1/indexer/{id}
```

**Response:** same shape as a single list entry above, with full field definitions.

#### Create Indexer

```http
POST /api/v1/indexer
```

**Request:**
```json
{
  "name": "ExampleTracker",
  "implementation": "Torznab",
  "protocol": "torrent",
  "enable": true,
  "priority": 25,
  "fields": [
    { "name": "baseUrl", "value": "https://example-tracker.example" },
    { "name": "apiKey", "value": "***" }
  ],
  "indexerProxyId": 1
}
```

Note: `indexerProxyId` is how an indexer is attached to a configured proxy (e.g., the FlareSolverr proxy — see Indexer Proxies API below) so that requests to this indexer are routed through challenge-solving first.

**Response:**
```json
{
  "id": 4,
  "name": "ExampleTracker",
  "enable": true,
  "protocol": "torrent",
  "indexerProxyId": 1,
  "added": "2026-10-23T12:00:00Z"
}
```

#### Update Indexer

```http
PUT /api/v1/indexer/{id}
```

**Request:** full indexer object with modified fields (e.g., `enable: false` to disable, or `indexerProxyId` to attach/detach a FlareSolverr proxy).

#### Delete Indexer

```http
DELETE /api/v1/indexer/{id}
```

**Response:** `200 OK`, empty body.

#### Test Indexer

```http
POST /api/v1/indexer/test
```

Validates an indexer configuration (connectivity, auth, categories) without saving it. If the indexer is behind Cloudflare and no proxy is attached, this call typically fails with a timeout or an HTML-challenge-page parse error — the standard symptom that a FlareSolverr proxy needs to be attached.

### Indexer Proxies API (FlareSolverr integration point)

#### List Indexer Proxies

```http
GET /api/v1/indexerproxy
```

**Response:**
```json
[
  {
    "id": 1,
    "name": "FlareSolverr",
    "implementation": "FlareSolverr",
    "fields": [
      { "name": "host", "value": "http://flaresolverr.media.svc.cluster.local:8191" },
      { "name": "requestTimeout", "value": 60 }
    ],
    "tags": []
  }
]
```

#### Create Indexer Proxy

```http
POST /api/v1/indexerproxy
```

Registers FlareSolverr as an indexer proxy that individual indexers can reference.

**Request:**
```json
{
  "name": "FlareSolverr",
  "implementation": "FlareSolverr",
  "configContract": "FlareSolverrSettings",
  "fields": [
    { "name": "host", "value": "http://flaresolverr.media.svc.cluster.local:8191" },
    { "name": "requestTimeout", "value": 60 }
  ],
  "tags": []
}
```

**Response:**
```json
{
  "id": 1,
  "name": "FlareSolverr",
  "implementation": "FlareSolverr",
  "fields": [
    { "name": "host", "value": "http://flaresolverr.media.svc.cluster.local:8191" },
    { "name": "requestTimeout", "value": 60 }
  ]
}
```

#### Test Indexer Proxy

```http
POST /api/v1/indexerproxy/test
```

Verifies Prowlarr can reach the configured FlareSolverr `host` URL and that FlareSolverr responds with a healthy session. A failure here almost always means either the FlareSolverr pod is down (`kubectl get pods -n media -l app.kubernetes.io/name=flaresolverr`) or the `host` field has the wrong in-cluster DNS name/port.

### Applications API (Radarr / Sonarr sync)

#### List Applications

```http
GET /api/v1/applications
```

**Response:**
```json
[
  {
    "id": 1,
    "name": "Radarr",
    "implementation": "Radarr",
    "syncLevel": "fullSync",
    "fields": [
      { "name": "prowlarrUrl", "value": "http://prowlarr.media.svc.cluster.local:9696" },
      { "name": "baseUrl", "value": "http://radarr.media.svc.cluster.local:7878" },
      { "name": "apiKey", "value": "***" }
    ]
  },
  {
    "id": 2,
    "name": "Sonarr",
    "implementation": "Sonarr",
    "syncLevel": "fullSync",
    "fields": [
      { "name": "prowlarrUrl", "value": "http://prowlarr.media.svc.cluster.local:9696" },
      { "name": "baseUrl", "value": "http://sonarr.media.svc.cluster.local:8989" },
      { "name": "apiKey", "value": "***" }
    ]
  }
]
```

#### Create Application

```http
POST /api/v1/applications
```

**Request:**
```json
{
  "name": "Radarr",
  "implementation": "Radarr",
  "configContract": "RadarrSettings",
  "syncLevel": "fullSync",
  "fields": [
    { "name": "prowlarrUrl", "value": "http://prowlarr.media.svc.cluster.local:9696" },
    { "name": "baseUrl", "value": "http://radarr.media.svc.cluster.local:7878" },
    { "name": "apiKey", "value": "***" }
  ]
}
```

#### Trigger Application Sync

```http
POST /api/v1/command
```

**Request:**
```json
{
  "name": "ApplicationIndexerSync"
}
```

Forces an immediate push of all matching indexers to every configured application (Radarr, Sonarr), rather than waiting for the next scheduled sync.

**Response:**
```json
{
  "id": 501,
  "name": "ApplicationIndexerSync",
  "status": "started",
  "queued": "2026-10-23T12:00:00Z"
}
```

### System API

#### Health Check

```http
GET /api/v1/health
```

**Response:**
```json
[
  {
    "source": "IndexerProxyCheck",
    "type": "warning",
    "message": "Indexer proxy FlareSolverr is unavailable",
    "wikiUrl": "https://wiki.servarr.com/prowlarr"
  }
]
```

An empty array means no active health warnings. This endpoint is the quickest way to confirm FlareSolverr connectivity programmatically — a populated `IndexerProxyCheck` entry means Prowlarr lost contact with FlareSolverr.

#### System Status

```http
GET /api/v1/system/status
```

**Response:**
```json
{
  "version": "2.6.5.0",
  "buildTime": "2026-09-15T00:00:00Z",
  "isDebug": false,
  "osName": "alpine",
  "runtimeVersion": "8.0.0",
  "startTime": "2026-10-22T07:04:33Z"
}
```

#### Ping

```http
GET /ping
```

Lightweight liveness check used by load balancers/monitoring; returns `{"status": "OK"}`.

## Error Handling

### Error Response Format

```json
{
  "message": "Indexer proxy test failed: connection refused",
  "description": "Could not connect to http://flaresolverr.media.svc.cluster.local:8191"
}
```

### Common HTTP Status Codes

| Code | Description |
|------|-------------|
| `200` | Success |
| `201` | Created |
| `400` | Validation error (malformed request body) |
| `401` | Missing/invalid API key |
| `404` | Resource not found |
| `500` | Internal server error (often surfaces indexer-proxy or indexer-parsing failures) |

## Rate Limiting

Prowlarr does not impose its own API rate limiting; however, downstream indexer sites may rate-limit search/test requests. Respect each indexer's documented request limits when scripting against this API, especially bulk `indexer/test` calls across many indexers.

## Client Examples

### Python

```python
import requests

BASE_URL = "https://prowlarr.conderhome.xyz/api/v1"
HEADERS = {"X-Api-Key": "***"}

# List indexers
indexers = requests.get(f"{BASE_URL}/indexer", headers=HEADERS).json()

# Trigger a full application sync (push indexers to Radarr/Sonarr)
requests.post(
    f"{BASE_URL}/command",
    headers=HEADERS,
    json={"name": "ApplicationIndexerSync"},
)

# Check health (flags FlareSolverr connectivity issues)
health = requests.get(f"{BASE_URL}/health", headers=HEADERS).json()
```

### curl

```bash
# List configured indexer proxies (confirm FlareSolverr is registered)
curl -s -H "X-Api-Key: ***" \
  "https://prowlarr.conderhome.xyz/api/v1/indexerproxy" | jq

# Trigger application sync
curl -s -X POST -H "X-Api-Key: ***" -H "Content-Type: application/json" \
  -d '{"name":"ApplicationIndexerSync"}' \
  "https://prowlarr.conderhome.xyz/api/v1/command"
```

---

*This API documentation covers the Prowlarr deployment in the conderhome homelab k3s cluster. For the full upstream API reference, see the Servarr wiki.*
