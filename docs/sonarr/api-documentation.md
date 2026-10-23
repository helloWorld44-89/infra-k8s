---
title: Sonarr API Documentation
date: 2026-10-23
author: Hermes Docs Agent
tags: [api, documentation, tv, media-automation]
---

# Sonarr API Documentation

## Overview

Sonarr exposes a comprehensive REST API (v3) for programmatic access to series management, episode tracking, queue/history, and system configuration. It is the same API the bundled web UI consumes internally, and the same API Prowlarr uses to sync indexers into Sonarr as an "Application".

**Base URL**: `https://sonarr.conderhome.xyz/api/v3` (external, via Traefik ingress)
**Internal Base URL**: `http://sonarr.media.svc.cluster.local:8989/api/v3` (in-cluster, via the k3s Service)

**Authentication**: API key (query parameter or header)

**Content-Type**: `application/json`

## Authentication

### Locating the API Key

The API key is generated automatically on first start and lives in two places:

1. **Web UI**: Settings → General → Security → API Key
2. **Config file on the PVC**: `/config/config.xml` inside the `sonarr-config` PersistentVolumeClaim, under the `<ApiKey>` element

```bash
# Read the API key directly from the live pod (via kubectl exec)
kubectl exec -n media deploy/sonarr -- cat /config/config.xml | grep -o '<ApiKey>[^<]*</ApiKey>'
```

### Using the API Key

Pass it as a header (preferred) or query parameter:

```bash
curl -H "X-Api-Key: ***" \
     "https://sonarr.conderhome.xyz/api/v3/series"

# Equivalent, query-parameter form
curl "https://sonarr.conderhome.xyz/api/v3/series?apikey=***"
```

## API Endpoints

### Series API

#### List Series

```http
GET /api/v3/series
```

Get all series currently tracked by Sonarr.

**Response:**
```json
[
  {
    "id": 1,
    "title": "Breaking Bad",
    "sortTitle": "breaking bad",
    "status": "ended",
    "overview": "A high school chemistry teacher...",
    "network": "AMC",
    "year": 2008,
    "path": "/tv/TV Shows/Breaking Bad",
    "qualityProfileId": 4,
    "languageProfileId": 1,
    "seasonFolder": true,
    "monitored": true,
    "seriesType": "standard",
    "tvdbId": 81189,
    "imdbId": "tt0903747",
    "seasons": [
      {
        "seasonNumber": 1,
        "monitored": true,
        "statistics": {
          "episodeFileCount": 7,
          "episodeCount": 7,
          "totalEpisodeCount": 7,
          "sizeOnDisk": 10737418240,
          "percentOfEpisodes": 100.0
        }
      }
    ],
    "statistics": {
      "seasonCount": 5,
      "episodeFileCount": 62,
      "episodeCount": 62,
      "totalEpisodeCount": 62,
      "sizeOnDisk": 96636764160,
      "percentOfEpisodes": 100.0
    }
  }
]
```

#### Get Series Details

```http
GET /api/v3/series/{id}
```

Get detailed information about a specific series.

**Response:** Same shape as a single list item above, with full `seasons` and `statistics` detail.

#### Add Series

```http
POST /api/v3/series
```

Add a new series to Sonarr's monitored library.

**Request:**
```json
{
  "title": "The Expanse",
  "qualityProfileId": 4,
  "languageProfileId": 1,
  "tvdbId": 275908,
  "rootFolderPath": "/tv/TV Shows",
  "seasonFolder": true,
  "monitored": true,
  "addOptions": {
    "searchForMissingEpisodes": true
  }
}
```

**Response:**
```json
{
  "id": 42,
  "title": "The Expanse",
  "tvdbId": 275908,
  "path": "/tv/TV Shows/The Expanse",
  "monitored": true,
  "status": "continuing"
}
```

#### Update Series

```http
PUT /api/v3/series/{id}
```

Update series settings (monitored state, quality profile, path).

**Request:**
```json
{
  "id": 42,
  "title": "The Expanse",
  "qualityProfileId": 6,
  "monitored": false
}
```

**Response:** Updated series object (same shape as above).

#### Delete Series

```http
DELETE /api/v3/series/{id}?deleteFiles=false
```

Remove a series from Sonarr. `deleteFiles=true` also deletes the episode files on disk.

**Response:** `200 OK`, empty body.

### Episode API

#### List Episodes

```http
GET /api/v3/episode?seriesId=1
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `seriesId` | integer | Filter by series (required for most calls) |
| `seasonNumber` | integer | Filter by season |
| `episodeIds` | array | Filter by specific episode IDs |
| `includeImages` | boolean | Include episode screenshot URLs |

**Response:**
```json
[
  {
    "id": 101,
    "seriesId": 1,
    "episodeFileId": 55,
    "seasonNumber": 1,
    "episodeNumber": 1,
    "title": "Pilot",
    "airDate": "2008-01-20",
    "airDateUtc": "2008-01-20T21:00:00Z",
    "hasFile": true,
    "monitored": true,
    "absoluteEpisodeNumber": 1
  }
]
```

#### Monitor/Unmonitor Episodes

```http
PUT /api/v3/episode/monitor
```

**Request:**
```json
{
  "episodeIds": [101, 102, 103],
  "monitored": true
}
```

**Response:** List of updated episode objects.

### Queue API

#### Get Download Queue

```http
GET /api/v3/queue
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `page` | integer | Page number (default: 1) |
| `pageSize` | integer | Items per page (default: 10) |
| `includeSeries` | boolean | Embed series detail |

**Response:**
```json
{
  "page": 1,
  "pageSize": 10,
  "totalRecords": 2,
  "records": [
    {
      "id": 9001,
      "seriesId": 1,
      "episodeId": 150,
      "title": "Breaking.Bad.S05E14.1080p.WEB-DL",
      "status": "downloading",
      "trackedDownloadStatus": "ok",
      "size": 2147483648,
      "sizeleft": 536870912,
      "downloadClient": "SABnzbd",
      "indexer": "NZBgeek",
      "estimatedCompletionTime": "2026-10-23T14:00:00Z"
    }
  ]
}
```

#### Remove From Queue

```http
DELETE /api/v3/queue/{id}?removeFromClient=true&blocklist=false
```

**Response:** `200 OK`, empty body.

### History API

#### List History

```http
GET /api/v3/history?page=1&pageSize=20&sortKey=date&sortDirection=descending
```

**Response:**
```json
{
  "page": 1,
  "pageSize": 20,
  "totalRecords": 500,
  "records": [
    {
      "id": 5001,
      "episodeId": 150,
      "seriesId": 1,
      "eventType": "downloadFolderImported",
      "date": "2026-10-22T03:12:00Z",
      "sourceTitle": "Breaking.Bad.S05E14.1080p.WEB-DL",
      "quality": {
        "quality": { "id": 7, "name": "WEBDL-1080p" }
      },
      "data": {
        "importedPath": "/tv/TV Shows/Breaking Bad/Season 05/Breaking Bad - S05E14 - Ozymandias.mkv"
      }
    }
  ]
}
```

### Command API

#### Trigger a Command

```http
POST /api/v3/command
```

Used to trigger ad-hoc operations: RSS sync, series search, episode search, rename, refresh.

**Request (RSS Sync):**
```json
{
  "name": "RssSync"
}
```

**Request (Series Search):**
```json
{
  "name": "SeriesSearch",
  "seriesId": 1
}
```

**Response:**
```json
{
  "id": 777,
  "name": "SeriesSearch",
  "status": "started",
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
  "id": 777,
  "name": "SeriesSearch",
  "status": "completed",
  "queued": "2026-10-23T12:00:00Z",
  "started": "2026-10-23T12:00:01Z",
  "ended": "2026-10-23T12:00:05Z",
  "duration": "00:00:04"
}
```

### Indexer & Download Client API

#### List Indexers

```http
GET /api/v3/indexer
```

In this deployment, indexers are synced in from Prowlarr via its "Applications" integration rather than added manually; this endpoint reflects whatever Prowlarr has pushed.

**Response:**
```json
[
  {
    "id": 1,
    "name": "NZBgeek (Prowlarr)",
    "implementation": "Newznab",
    "protocol": "usenet",
    "enableRss": true,
    "enableAutomaticSearch": true,
    "enableInteractiveSearch": true,
    "priority": 25
  }
]
```

#### List Download Clients

```http
GET /api/v3/downloadclient
```

**Response:**
```json
[
  {
    "id": 1,
    "name": "SABnzbd",
    "implementation": "Sabnzbd",
    "protocol": "usenet",
    "enable": true,
    "priority": 1
  }
]
```

### System API

#### Health Check

```http
GET /api/v3/health
```

**Response:**
```json
[
  {
    "source": "RssSyncCheck",
    "type": "warning",
    "message": "No available indexers. At least one indexer is required for RSS to work.",
    "wikiUrl": "https://wiki.servarr.com/sonarr/system#rss-sync"
  }
]
```

An empty array (`[]`) indicates a fully healthy instance.

#### Unauthenticated Ping

```http
GET /ping
```

**Response:**
```json
{
  "status": "OK"
}
```

Useful as a Kubernetes liveness/readiness probe target that doesn't require an API key.

#### System Status

```http
GET /api/v3/system/status
```

**Response:**
```json
{
  "version": "4.0.20.2706",
  "buildTime": "2026-09-15T00:00:00Z",
  "isDebug": false,
  "isProduction": true,
  "isDocker": true,
  "osName": "alpine",
  "runtimeVersion": "8.0.8",
  "appData": "/config",
  "startupPath": "/app/bin"
}
```

### Webhooks (Connect / Notifications)

Sonarr does not call outbound "webhooks" by that name in the API — instead, "Connections" configured under Settings → Connect fire HTTP(S) webhook requests (and/or Discord, Slack, etc.) on events.

**Example payload (On Import):**
```json
{
  "eventType": "Download",
  "series": {
    "id": 1,
    "title": "Breaking Bad",
    "tvdbId": 81189
  },
  "episodes": [
    {
      "episodeNumber": 14,
      "seasonNumber": 5,
      "title": "Ozymandias"
    }
  ],
  "episodeFile": {
    "relativePath": "Season 05/Breaking Bad - S05E14 - Ozymandias.mkv",
    "quality": "WEBDL-1080p"
  },
  "isUpgrade": false
}
```

Configure under Settings → Connect → Add Connection → Webhook, set the target URL, select trigger events (On Grab, On Import, On Upgrade, On Rename, On Series Delete, On Health Issue, etc.).

## Error Handling

### Error Response Format

```json
{
  "message": "Series could not be found"
}
```

Validation errors return a list:
```json
[
  {
    "propertyName": "RootFolderPath",
    "errorMessage": "Root folder path does not exist",
    "severity": "error"
  }
]
```

### Common HTTP Status Codes

| Code | Description |
|------|-------------|
| `200` | Success |
| `201` | Created (e.g., series/episode added) |
| `400` | Validation error — see response body for field-level detail |
| `401` | Missing or invalid API key |
| `404` | Resource not found |
| `500` | Internal server error |

## Rate Limiting

Sonarr does not enforce application-level API rate limiting on its own `/api/v3` endpoints by default. Be considerate with automated polling (e.g., queue/history pollers should use a sane interval, 10-30s, rather than tight loops) since all requests hit the same single SQLite-backed process.

## Client Libraries

### Python

No official SDK ships with Sonarr; the community `pyarr` package wraps `/api/v3`:

```python
from pyarr import SonarrAPI

sonarr = SonarrAPI("https://sonarr.conderhome.xyz", "***")

# List series
series = sonarr.get_series()

# Trigger an RSS sync
sonarr.post_command("RssSync")

# Search for a specific series
sonarr.post_command("SeriesSearch", seriesId=1)
```

### curl / shell

```bash
# Add series by TVDB ID
curl -X POST "https://sonarr.conderhome.xyz/api/v3/series" \
  -H "X-Api-Key: ***" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "The Expanse",
    "qualityProfileId": 4,
    "languageProfileId": 1,
    "tvdbId": 275908,
    "rootFolderPath": "/tv/TV Shows",
    "monitored": true,
    "addOptions": {"searchForMissingEpisodes": true}
  }'
```

## Integration Notes

- **Prowlarr → Sonarr**: Prowlarr calls this same `/api/v3` surface (specifically the indexer config endpoints) using an API key you generate in Sonarr and paste into Prowlarr's "Sonarr" Application entry. This is how indexer management stays centralized in Prowlarr.
- **FlareSolverr**: Never called directly by Sonarr. It's wired into specific Prowlarr-managed indexers as an HTTP proxy for Cloudflare-protected trackers; Sonarr is unaware of FlareSolverr's existence.
- **Radarr**: A structurally identical API (`/api/v3`) exists on the Radarr instance (`radarr.conderhome.xyz`) for movies — concepts map 1:1 (series→movie, episode→n/a) but the two APIs are entirely independent instances.

---

*This API documentation covers Sonarr v4 (`linuxserver/sonarr:4.0.20`) as deployed in the `media` namespace. For the complete, authoritative endpoint reference, see the official Sonarr API docs at https://sonarr.tv or the in-app Swagger UI at `/api/v3/docs` (if enabled) or the project wiki.*
