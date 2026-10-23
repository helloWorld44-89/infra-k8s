---
title: Immich API Documentation
date: 2026-10-23
author: Hermes Docs Agent
tags: [api, documentation, photos, openapi]
---

# Immich API Documentation

## Overview

Immich ships a complete, auto-generated REST API covering assets, albums, users, search, and administration. The server generates an OpenAPI 3.0 specification at runtime, so the exact endpoint list always matches the deployed version (`v3.2.2` in this cluster).

**Base URL (internal)**: `http://immich-server.media.svc.cluster.local:2283/api`

**Base URL (external)**: `https://immich.conderhome.xyz/api`

**OpenAPI / Swagger**: the running server exposes a live spec and interactive docs:
- Swagger UI: `https://immich.conderhome.xyz/docs` (`/api/docs` on some versions)
- Raw OpenAPI JSON: `https://immich.conderhome.xyz/api-json` (or `/docs/json` depending on build)
- The spec is also published in the upstream repo at `immich-app/immich` under `open-api/`.

**Authentication**:
- Session cookie (browser/Web UI login)
- Bearer JWT access token (`Authorization: Bearer <token>`)
- API Key (`x-api-key` header) — recommended for scripts/automation

**Content-Type**: `application/json` (file uploads use `multipart/form-data`)

## Authentication

### Logging In

```bash
curl -X POST "https://immich.conderhome.xyz/api/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"email": "user@example.com", "password": "***"}'
```

Response includes `accessToken` (JWT) to use as a Bearer token for subsequent requests.

### Creating an API Key

API keys are created from the Web UI (Account Settings → API Keys) or programmatically once authenticated:

```bash
curl -X POST "https://immich.conderhome.xyz/api/api-keys" \
  -H "Authorization: Bearer <access-token>" \
  -H "Content-Type: application/json" \
  -d '{"name": "homelab-automation"}'
```

### Using an API Key

```bash
curl -H "x-api-key: ***" \
     "https://immich.conderhome.xyz/api/server/version"
```

## API Endpoints

### Server / System

#### Get Server Version

```http
GET /api/server/version
```

Returns the running Immich server version.

**Response:**
```json
{
  "major": 3,
  "minor": 2,
  "patch": 2,
  "prerelease": null
}
```

#### Get Server Configuration

```http
GET /api/server/config
```

Returns public server configuration (feature flags, trash retention, OAuth button text, map tile URLs, etc.).

**Response (excerpt from this deployment):**
```json
{
  "loginPageMessage": "",
  "trashDays": 30,
  "userDeleteDelay": 7,
  "oauthButtonText": "Login with OAuth",
  "isInitialized": true,
  "isOnboarded": true,
  "publicUsers": true,
  "mapDarkStyleUrl": "https://tiles.immich.cloud/v1/style/dark.json",
  "mapLightStyleUrl": "https://tiles.immich.cloud/v1/style/light.json",
  "maintenanceMode": false,
  "minFaces": 3
}
```

#### Server Statistics

```http
GET /api/server/statistics
```

Admin-only. Returns total photos, videos, and usage counts across all users.

#### Server Storage

```http
GET /api/server/storage
```

Returns disk usage statistics for the server's storage volume.

### Assets API

#### Upload Asset

```http
POST /api/assets
```

Uploads a new photo or video asset. `multipart/form-data` with the file plus metadata fields.

**Request (form fields):**
```
assetData: <binary file>
deviceAssetId: "unique-device-id"
deviceId: "my-device"
fileCreatedAt: "2026-10-23T10:00:00.000Z"
fileModifiedAt: "2026-10-23T10:00:00.000Z"
```

**Response:**
```json
{
  "id": "c1e2f3a4-5678-90ab-cdef-1234567890ab",
  "status": "created",
  "duplicateId": null
}
```

#### Get Asset Info

```http
GET /api/assets/{id}
```

**Response:**
```json
{
  "id": "c1e2f3a4-5678-90ab-cdef-1234567890ab",
  "deviceAssetId": "unique-device-id",
  "ownerId": "user-id-123",
  "type": "IMAGE",
  "originalFileName": "IMG_0001.jpg",
  "originalPath": "/usr/src/app/upload/library/user/2026/2026-10-23/IMG_0001.jpg",
  "fileCreatedAt": "2026-10-23T10:00:00.000Z",
  "fileModifiedAt": "2026-10-23T10:00:00.000Z",
  "exifInfo": {
    "make": "Apple",
    "model": "iPhone 15 Pro",
    "exifImageWidth": 4032,
    "exifImageHeight": 3024,
    "latitude": 37.7749,
    "longitude": -122.4194
  },
  "smartInfo": {
    "tags": ["outdoor", "sky", "mountain"],
    "objects": ["person"]
  },
  "isFavorite": false,
  "isArchived": false,
  "duration": null
}
```

#### Download Original Asset

```http
GET /api/assets/{id}/original
```

Streams the original file bytes.

#### Delete Assets

```http
DELETE /api/assets
```

**Request:**
```json
{
  "ids": ["c1e2f3a4-5678-90ab-cdef-1234567890ab"],
  "force": false
}
```

Soft-deletes (moves to trash) unless `force: true`.

### Search API

#### Smart Search

```http
POST /api/search/smart
```

Natural-language / CLIP-powered search over the library (routed through `immich-machine-learning`).

**Request:**
```json
{
  "query": "dog playing in the snow",
  "size": 50
}
```

**Response:**
```json
{
  "assets": {
    "total": 3,
    "items": [
      {
        "id": "a1b2c3d4-...",
        "originalFileName": "IMG_0452.jpg",
        "type": "IMAGE"
      }
    ]
  }
}
```

#### Metadata Search

```http
POST /api/search/metadata
```

Filter by structured fields: date range, camera make/model, city, tags, favorites, archived status.

### Albums API

#### List Albums

```http
GET /api/albums
```

**Response:**
```json
[
  {
    "id": "album-id-1",
    "albumName": "Summer 2026",
    "assetCount": 128,
    "createdAt": "2026-06-01T00:00:00.000Z",
    "shared": false
  }
]
```

#### Create Album

```http
POST /api/albums
```

**Request:**
```json
{
  "albumName": "Summer 2026",
  "assetIds": ["c1e2f3a4-5678-90ab-cdef-1234567890ab"]
}
```

#### Add Assets to Album

```http
PUT /api/albums/{id}/assets
```

**Request:**
```json
{
  "ids": ["c1e2f3a4-5678-90ab-cdef-1234567890ab"]
}
```

### People / Faces API

#### List Recognized People

```http
GET /api/people
```

**Response:**
```json
{
  "total": 12,
  "people": [
    {
      "id": "person-id-1",
      "name": "Alex",
      "thumbnailPath": "/api/people/person-id-1/thumbnail",
      "isHidden": false
    }
  ]
}
```

#### Get Assets for a Person

```http
GET /api/people/{id}/assets
```

### Users API

#### Get Current User

```http
GET /api/users/me
```

**Response:**
```json
{
  "id": "user-id-123",
  "email": "user@example.com",
  "name": "Alex",
  "isAdmin": true,
  "quotaSizeInBytes": null,
  "quotaUsageInBytes": 548900000000,
  "createdAt": "2026-01-15T00:00:00.000Z"
}
```

### Jobs API (Admin)

#### Get Job Status

```http
GET /api/jobs
```

Returns queue depth and status for each background job type (thumbnail generation, metadata extraction, face detection, smart search indexing, library scan, storage template migration).

**Response (excerpt):**
```json
{
  "thumbnailGeneration": {
    "queueStatus": { "isActive": false, "isPaused": false },
    "jobCounts": { "active": 0, "completed": 1540, "failed": 0, "delayed": 0, "waiting": 0 }
  },
  "smartSearch": {
    "queueStatus": { "isActive": true, "isPaused": false },
    "jobCounts": { "active": 1, "completed": 980, "failed": 0, "delayed": 0, "waiting": 4 }
  }
}
```

#### Trigger a Job

```http
POST /api/jobs/{jobName}
```

**Request:**
```json
{
  "command": "start",
  "force": false
}
```

## Error Handling

### Error Response Format

```json
{
  "message": "Asset not found",
  "error": "Not Found",
  "statusCode": 404
}
```

### Common HTTP Status Codes

| Status | Meaning |
|--------|---------|
| `400` | Bad Request — validation failed |
| `401` | Unauthorized — missing/invalid token or API key |
| `403` | Forbidden — insufficient permissions (non-admin calling admin route) |
| `404` | Not Found — asset/album/user does not exist |
| `409` | Conflict — duplicate asset / resource conflict |
| `500` | Internal Server Error |

## Rate Limiting

Immich does not enforce aggressive built-in API rate limiting for self-hosted deployments by default; the practical limits in this cluster are the `immich-server` pod's CPU/memory and the shared-postgres connection pool. Heavy bulk operations (e.g. mass re-scan, library sync) should be run outside peak usage hours.

## Client Libraries

### Official Mobile/Desktop Apps

Immich's own mobile apps (iOS/Android) and CLI (`immich` npm package) use this same API and are the primary consumers in this homelab, pointed at `https://immich.conderhome.xyz`.

### Immich CLI

```bash
npm install -g @immich/cli

immich login https://immich.conderhome.xyz <api-key>
immich upload --recursive /path/to/photos
```

### Direct curl Example (list recent assets)

```bash
curl -s "https://immich.conderhome.xyz/api/assets" \
  -H "x-api-key: ***" | jq '.[] | {id, originalFileName, fileCreatedAt}'
```

## Webhooks

Immich does not have a general-purpose outbound webhook system as of `v3.2.2`; integration is primarily done by polling the Jobs/Assets/Search APIs or via the CLI for scripted sync/upload workflows.

---

*This API documentation is part of the Immich deployment docs for this homelab cluster. For the authoritative, version-matched spec, always check the live Swagger UI at `https://immich.conderhome.xyz/docs` on the running server.*
