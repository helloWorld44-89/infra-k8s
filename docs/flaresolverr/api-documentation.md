---
title: FlareSolverr API Documentation
date: 2026-10-23
author: Hermes Docs Agent
tags: [api, documentation, flaresolverr, proxy, internal]
---

# FlareSolverr API Documentation

## Overview

FlareSolverr exposes a single, unauthenticated internal JSON HTTP API. It is **not a public or end-user API** — it is consumed only by Prowlarr inside this cluster, which was configured with it as a FlareSolverr-type "Indexer Proxy." There is no OpenAPI/Swagger UI, no API keys, and no versioned `/api/...` namespace of resources (books, users, etc.) the way a typical application API would have — the entire surface is one command endpoint.

**Base URL (in-cluster only)**: `http://flaresolverr.media.svc.cluster.local:8191`

**Authentication**: None. Access control is purely network-level (ClusterIP Service, no Ingress — unreachable from outside the cluster).

**Content-Type**: `application/json`

## The `/v1` Endpoint

```http
POST /v1
```

Every interaction with FlareSolverr — fetching a URL, submitting a form, or managing a persistent session — goes through this single endpoint. The `cmd` field in the request body selects the operation.

### Common Request Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `cmd` | string | yes | Operation: `request.get`, `request.post`, `sessions.create`, `sessions.list`, `sessions.destroy` |
| `url` | string | for `request.*` | Target URL to fetch through the solved browser session |
| `session` | string | no | Reuse an existing persistent session instead of a one-off browser launch |
| `maxTimeout` | integer | no | Max milliseconds to wait for the challenge to resolve (default: 60000) |
| `cookies` | array | no | Cookies to seed into the browser before loading the page |
| `postData` | string | for `request.post` | URL-encoded form body to submit |
| `proxy` | object | no | Upstream proxy for FlareSolverr's own outbound Chromium traffic (`url`, `username`, `password`) |

### `request.get` — Solve a Challenge and Fetch a Page

This is the operation Prowlarr issues for every search/grab against a Cloudflare-protected indexer.

**Request:**
```json
{
  "cmd": "request.get",
  "url": "https://example-indexer.tld/search?q=some+release",
  "maxTimeout": 60000
}
```

**Response (success):**
```json
{
  "status": "ok",
  "message": "",
  "startTimestamp": 1793012345000,
  "endTimestamp": 1793012351200,
  "version": "3.3.21",
  "solution": {
    "url": "https://example-indexer.tld/search?q=some+release",
    "status": 200,
    "headers": {},
    "response": "<!doctype html>...<html>...full page HTML...</html>",
    "cookies": [
      {
        "name": "cf_clearance",
        "value": "***",
        "domain": ".example-indexer.tld",
        "path": "/",
        "expires": 1793098751,
        "secure": true,
        "httpOnly": true
      }
    ],
    "userAgent": "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 ..."
  }
}
```

**Response (challenge not solved / timeout):**
```json
{
  "status": "error",
  "message": "Error solving the challenge. Timeout after 60000ms.",
  "startTimestamp": 1793012345000,
  "endTimestamp": 1793012405000,
  "version": "3.3.21"
}
```

### `request.post` — Solve a Challenge and Submit a Form

```json
{
  "cmd": "request.post",
  "url": "https://example-indexer.tld/login",
  "postData": "username=foo&password=bar",
  "maxTimeout": 60000
}
```

Response shape is identical to `request.get`'s `solution` object.

### `sessions.create` — Create a Persistent Browser Session

Avoids re-solving the challenge on every call by keeping a Chromium context (and its cookies) alive in the FlareSolverr pod between requests.

**Request:**
```json
{
  "cmd": "sessions.create",
  "session": "prowlarr-indexer-x"
}
```

**Response:**
```json
{
  "status": "ok",
  "message": "Session created successfully.",
  "session": "prowlarr-indexer-x"
}
```

> Sessions are held in-memory in the single FlareSolverr pod. A pod restart (deploy, node drain, OOM) silently drops all sessions — any indexer relying on `session` will transparently fall back to a fresh challenge-solve on its next call rather than erroring, but that call will be noticeably slower.

### `sessions.list` — List Active Sessions

**Request:**
```json
{ "cmd": "sessions.list" }
```

**Response:**
```json
{
  "status": "ok",
  "message": "",
  "sessions": ["prowlarr-indexer-x"]
}
```

### `sessions.destroy` — Destroy a Session

**Request:**
```json
{
  "cmd": "sessions.destroy",
  "session": "prowlarr-indexer-x"
}
```

**Response:**
```json
{
  "status": "ok",
  "message": "The session has been removed."
}
```

## Error Handling

FlareSolverr does not use HTTP status codes to signal failure — it almost always returns `200 OK` with a JSON body whose top-level `status` field is `"ok"` or `"error"`. Callers (Prowlarr) must inspect `status`/`message`, not the HTTP status line.

| `status` | Meaning |
|----------|---------|
| `ok` | Challenge solved (or no challenge present); `solution` populated |
| `error` | Challenge could not be solved — timeout, unsupported CAPTCHA type (hCaptcha/reCAPTCHA; `CAPTCHA_SOLVER` is `none` in this deployment so these always fail), or the target site is unreachable from the pod |

There is no rate limiting, no API key tiers, and no webhook/callback mechanism — this is a synchronous request/response proxy only.

## How Prowlarr Uses This API

Prowlarr is the only configured caller. Inside Prowlarr's **Settings → Indexers → Indexer Proxies**, a "FlareSolverr" proxy is defined pointing at `http://flaresolverr.media.svc.cluster.local:8191`, then attached to the specific indexer definitions that are Cloudflare-protected. Prowlarr internally issues `request.get`/`request.post` calls exactly as documented above — there is no separate Prowlarr-side API surface to configure beyond that proxy URL and timeout.

There is no SDK, no client library, and no other integration path — if you need to reach FlareSolverr directly for debugging, use `kubectl exec`/`port-forward` and `curl` against `/v1` as shown above.

---

*This API documentation is part of the FlareSolverr internal service running in this cluster. FlareSolverr itself has no public API docs site beyond its upstream GitHub README — this file documents how it's actually used in this deployment.*
