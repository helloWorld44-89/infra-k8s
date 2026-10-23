---
title: Radarr API Documentation
date: 2026-10-23
author: Hermes Docs Agent
tags: [api, documentation, movies, radarr]
---

# Radarr API Documentation

## Overview

Radarr exposes a comprehensive REST API (v3) for programmatic access to movie management, searching, queue control, and system status. The API backs Radarr's own web UI and is also how Prowlarr pushes indexer configuration into Radarr.

**Base URL**: `https://radarr.conderhome.xyz/api/v3` (external, via Traefik ingress)
**Internal Base URL**: `http://radarr.media.svc.cluster.local:7878/api/v3` (in-cluster, service DNS)

**Authentication**: API key (query parameter or header)

**Content-Type**: `application/json`

## Authentication

### Locating the API Key

The API key is generated automatically on first start and is required on every API call. It can be found in two places:

1. **Web UI**: Settings → General → Security → API Key
2. **Config file on the PVC**: `/config/config.xml`, inside the `<ApiKey>` element

```bash
# Via kubectl (redact before sharing — treat this as a secret)
kubectl exec -n media deploy/radarr -- cat /config/config.xml | grep ApiKey
```

The key is a long-lived shared secret — rotate it from Settings → General if it is ever exposed, which invalidates all existing integrations (Prowlarr sync, scripts, Homarr/dashboard widgets, etc.) until they're updated.

### Using the API Key

Radarr accepts the key either as a header or as a query parameter:

```bash
# Header (preferred)
curl -H "X-Api-Key: ***" \
     "https://radarr.conderhome.xyz/api/v3/movie"

# Query parameter
curl "https://radarr.conderhome.xyz/api/v3/movie?apikey=***"
```

## API Endpoints

### Movies API

#### List Movies

```http
GET /api/v3/movie
```

Get all movies currently in the library.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `tmdbId` | integer | Filter by TMDb ID |
| `excludeLocalCovers` | boolean | Exclude local cover art paths from response |

**Response:**
```json
[
  {
    "id": 1,
    "title": "Dune: Part Two",
    "originalTitle": "Dune: Part Two",
    "year": 2024,
    "tmdbId": 693134,
    "imdbId": "tt15239678",
    "titleSlug": "dune-part-two-693134",
    "qualityProfileId": 4,
    "monitored": true,
    "minimumAvailability": "released",
    "isAvailable": true,
    "folderName": "/movies/Dune Part Two (2024)",
    "path": "/movies/Dune Part Two (2024)",
    "sizeOnDisk": 21474836480,
    "status": "released",
    "hasFile": true,
    "movieFile": {
      "id": 1,
      "relativePath": "Dune Part Two (2024) Bluray-1080p.mkv",
      "size": 21474836480,
      "quality": {
        "quality": {
          "id": 7,
          "name": "Bluray-1080p"
        }
      }
    },
    "added": "2026-03-01T10:00:00Z",
    "ratings": {
      "tmdb": {
        "value": 8.2,
        "votes": 4500
      }
    }
  }
]
```

#### Get Movie Details

```http
GET /api/v3/movie/{id}
```

Get detailed information about a specific movie.

**Response:** same shape as the list item above, scoped to one movie.

#### Add Movie

```http
POST /api/v3/movie
```

Add a new movie to the library (usually preceded by a `GET /api/v3/movie/lookup?term=` call to resolve the TMDb ID).

**Request:**
```json
{
  "title": "New Movie",
  "tmdbId": 123456,
  "qualityProfileId": 4,
  "rootFolderPath": "/movies",
  "monitored": true,
  "minimumAvailability": "released",
  "addOptions": {
    "searchForMovie": true
  }
}
```

**Response:**
```json
{
  "id": 2,
  "title": "New Movie",
  "tmdbId": 123456,
  "qualityProfileId": 4,
  "monitored": true,
  "path": "/movies/New Movie (2026)",
  "added": "2026-10-23T12:00:00Z"
}
```

#### Update Movie

```http
PUT /api/v3/movie/{id}
```

Update monitoring status, quality profile, or other movie settings. Requires the full movie resource (fetch via GET first, modify, then PUT).

**Response:** the updated movie resource.

#### Delete Movie

```http
DELETE /api/v3/movie/{id}
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `deleteFiles` | boolean | Also delete the movie file(s) from disk |
| `addImportExclusion` | boolean | Prevent the movie from being re-added automatically |

**Response:** `200 OK` with empty body.

#### Movie Lookup

```http
GET /api/v3/movie/lookup?term={search}
```

Search Radarr's metadata provider (TMDb-backed) for movies to add.

**Response:**
```json
[
  {
    "title": "Example Movie",
    "year": 2026,
    "tmdbId": 999999,
    "imdbId": "tt0000000",
    "overview": "Description...",
    "remotePoster": "https://image.tmdb.org/..."
  }
]
```

### Queue API

#### Get Queue

```http
GET /api/v3/queue
```

List items currently downloading or awaiting import.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `page` | integer | Page number |
| `pageSize` | integer | Items per page |
| `includeMovie` | boolean | Embed full movie object |

**Response:**
```json
{
  "page": 1,
  "pageSize": 20,
  "totalRecords": 1,
  "records": [
    {
      "id": 100,
      "movieId": 1,
      "title": "Dune: Part Two",
      "status": "downloading",
      "trackedDownloadStatus": "ok",
      "size": 21474836480,
      "sizeleft": 5368709120,
      "estimatedCompletionTime": "2026-10-23T14:00:00Z",
      "downloadClient": "qBittorrent",
      "protocol": "torrent"
    }
  ]
}
```

#### Remove Queue Item

```http
DELETE /api/v3/queue/{id}
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `removeFromClient` | boolean | Also remove from the download client |
| `blocklist` | boolean | Blocklist the release so it isn't grabbed again |

### Commands API

#### Trigger a Command

```http
POST /api/v3/command
```

Radarr's command system drives searches, scans, and refreshes.

**Request (search for a specific movie):**
```json
{
  "name": "MoviesSearch",
  "movieIds": [1]
}
```

**Other common commands:** `RefreshMovie`, `RescanMovie`, `RssSync`, `Backup`, `MissingMoviesSearch`

**Response:**
```json
{
  "id": 42,
  "name": "MoviesSearch",
  "status": "queued",
  "queued": "2026-10-23T12:00:00Z",
  "trigger": "manual"
}
```

#### Get Command Status

```http
GET /api/v3/command/{id}
```

**Response:**
```json
{
  "id": 42,
  "name": "MoviesSearch",
  "status": "completed",
  "queued": "2026-10-23T12:00:00Z",
  "started": "2026-10-23T12:00:01Z",
  "ended": "2026-10-23T12:00:15Z",
  "duration": "00:00:14"
}
```

### History API

#### Get History

```http
GET /api/v3/history
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `page` | integer | Page number |
| `pageSize` | integer | Items per page |
| `eventType` | string | `grabbed`, `downloadFolderImported`, `downloadFailed`, etc. |
| `movieId` | integer | Filter by movie |

**Response:**
```json
{
  "page": 1,
  "pageSize": 20,
  "totalRecords": 50,
  "records": [
    {
      "id": 500,
      "movieId": 1,
      "sourceTitle": "Dune.Part.Two.2024.Bluray-1080p",
      "eventType": "downloadFolderImported",
      "date": "2026-10-20T15:51:52Z",
      "data": {
        "importedPath": "/movies/Dune Part Two (2024)/Dune Part Two (2024) Bluray-1080p.mkv"
      }
    }
  ]
}
```

### Indexer API

#### List Indexers

```http
GET /api/v3/indexer
```

Lists all configured indexers — including those synced in from Prowlarr.

**Response:**
```json
[
  {
    "id": 1,
    "name": "ExampleIndexer (Prowlarr)",
    "implementation": "Newznab",
    "enableRss": true,
    "enableAutomaticSearch": true,
    "enableInteractiveSearch": true,
    "protocol": "usenet",
    "priority": 25
  }
]
```

### Quality Profile API

#### List Quality Profiles

```http
GET /api/v3/qualityprofile
```

**Response:**
```json
[
  {
    "id": 4,
    "name": "HD-1080p",
    "upgradeAllowed": true,
    "cutoff": 7,
    "items": [
      {
        "quality": { "id": 7, "name": "Bluray-1080p" },
        "allowed": true
      }
    ]
  }
]
```

### Root Folder API

#### List Root Folders

```http
GET /api/v3/rootfolder
```

**Response:**
```json
[
  {
    "id": 1,
    "path": "/movies",
    "accessible": true,
    "freeSpace": 536870912000,
    "totalSpace": 1073741824000
  }
]
```

### System API

#### System Status

```http
GET /api/v3/system/status
```

**Response:**
```json
{
  "version": "6.4.4",
  "buildTime": "2026-09-15T00:00:00Z",
  "isDebug": false,
  "isProduction": true,
  "osName": "linux",
  "runtimeVersion": "8.0.0",
  "startTime": "2026-10-20T15:51:55Z",
  "instanceName": "Radarr"
}
```

#### Health Check

```http
GET /api/v3/health
```

Returns a list of active health warnings/errors (empty array when everything is healthy).

**Response:**
```json
[
  {
    "source": "IndexerLongTermStatusCheck",
    "type": "warning",
    "message": "Indexers unavailable due to failures: ExampleIndexer",
    "wikiUrl": "https://wiki.servarr.com/radarr/system#indexers-are-unavailable"
  }
]
```

### Webhooks (Connect / Notifications)

Radarr pushes events to configured "Connect" notification targets (Settings → Connect) rather than exposing an inbound webhook receiver. Supported trigger events include:

- `onGrab` — a release has been grabbed and sent to the download client
- `onDownload` — a movie file has been imported
- `onUpgrade` — an existing file was replaced with a higher-quality one
- `onRename` — files were renamed
- `onHealthIssue` — a health check failure occurred
- `onApplicationUpdate` — Radarr itself updated

**Example outbound payload (`onDownload`):**
```json
{
  "movie": {
    "id": 1,
    "title": "Dune: Part Two",
    "year": 2024,
    "tmdbId": 693134
  },
  "movieFile": {
    "relativePath": "Dune Part Two (2024) Bluray-1080p.mkv",
    "quality": { "quality": { "name": "Bluray-1080p" } }
  },
  "isUpgrade": false,
  "eventType": "Download"
}
```

## Error Handling

### Error Response Format

```json
{
  "message": "Movie could not be found",
  "description": null
}
```

Validation errors return an array of field-level problems:

```json
[
  {
    "propertyName": "RootFolderPath",
    "errorMessage": "Root folder does not exist",
    "severity": "error"
  }
]
```

### Common HTTP Status Codes

| Status | Meaning |
|--------|---------|
| `200` | Success |
| `201` | Resource created |
| `400` | Validation error |
| `401` | Missing/invalid API key |
| `404` | Resource not found |
| `500` | Internal server error |

## Rate Limiting

Radarr does not enforce API rate limiting by default — it is designed for single-tenant, low-volume automation use (Prowlarr sync, scripts, dashboards). Avoid polling high-frequency endpoints (e.g. `/api/v3/queue`) faster than once every few seconds to keep load reasonable on the single-replica instance.

## Integration Notes

### Prowlarr → Radarr Sync

Prowlarr is configured as an "Application" pointing at this Radarr instance's base URL and API key. Prowlarr uses the same `POST/PUT /api/v3/indexer` endpoints shown above to push and keep indexer configuration in sync — no manual indexer setup is required in Radarr once the Prowlarr application link is established.

### Scripting Example (bash)

```bash
API_KEY="$(kubectl exec -n media deploy/radarr -- grep -oP '(?<=<ApiKey>)[^<]+' /config/config.xml)"
BASE="https://radarr.conderhome.xyz/api/v3"

# List missing monitored movies
curl -s -H "X-Api-Key: $API_KEY" "$BASE/movie" | jq '[.[] | select(.monitored and (.hasFile|not))]'

# Trigger a search for all missing movies
curl -s -X POST -H "X-Api-Key: $API_KEY" -H "Content-Type: application/json" \
  -d '{"name":"MissingMoviesSearch"}' "$BASE/command"
```

---

*This API documentation covers Radarr v3 (app version 6.4.4) as deployed in this cluster. For the exhaustive upstream reference, see the Servarr wiki's Radarr API section.*
