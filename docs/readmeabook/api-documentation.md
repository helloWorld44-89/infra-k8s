---
title: ReadMeABook API Documentation
date: 2026-10-23
author: Hermes Docs Agent
tags: [api, documentation, audiobook, kubernetes]
---

# ReadMeABook API Documentation

## Overview

ReadMeABook is an end-user web application (audiobook request/acquisition engine), not a published API product. There is no public, documented REST API surface intended for third-party integration — unlike AudioFile, the upstream project does not ship an OpenAPI/Swagger spec or an SDK. The application does have an internal API that the web UI itself calls, and a Kubernetes-facing health endpoint, both documented below from what is actually configured and observed in this deployment.

**Base URL (internal, cluster)**: `http://readmeabook.media.svc.cluster.local:3030`
**Base URL (external)**: `https://audiobook.conderhome.xyz`

## Health Endpoint

This is the only endpoint this deployment relies on programmatically — it backs both the liveness and readiness probes defined in the HelmRelease (`apps/media/readmeabook/helmrelease.yml`).

```http
GET /api/health
```

**Port**: `3030` (container), exposed via the `readmeabook` ClusterIP service on the `http` port.

**Used by:**
- Liveness probe — 60s initial delay, every 30s, 3 consecutive failures triggers a pod restart
- Readiness probe — 5s initial delay, every 10s, gates traffic via the Service/Ingress

**Manual check from inside the cluster:**
```bash
kubectl exec -n media deploy/readmeabook -- wget -qO- http://localhost:3030/api/health
```

**Manual check via ingress:**
```bash
curl -s https://audiobook.conderhome.xyz/api/health
```

No response schema was captured for this environment; treat any `200 OK` as healthy and any non-2xx / timeout as unhealthy, matching the probe's `httpGet` semantics.

## Internal/Admin Surface (not a supported public API)

The web app exposes its own internal endpoints under the app (e.g. request creation, search, settings, BookDate swipe actions, admin approval actions) that the bundled frontend calls directly. These are implementation details of the UI, not a documented/stable integration API:

- No API key issuance flow is exposed to end users in this deployment
- No webhook configuration was found wired up in this deployment's settings
- No SDKs are published upstream for this project

**Guidance for this homelab**: if programmatic access is ever needed (e.g. scripting a bulk import of requests), inspect the browser's network tab while using the web UI to reverse the internal endpoints — do not assume stability across upstream releases, since `image.tag: latest` means the running version can change on any pod restart (Flux reconciles the HelmRelease on a 1h interval and the image pull policy is `Always`).

## Upstream Integration APIs (consumed by ReadMeABook, not exposed by it)

ReadMeABook is itself an API *client* to several other services configured through its settings pages, not inspected further here:

- **Prowlarr** — indexer search API
- **qBittorrent** — torrent client Web API
- **SABnzbd** — usenet client API
- **Plex** / **Audiobookshelf** — library server APIs, for import/scan triggering
- **Audible** — metadata/search backend for book lookups
- **OpenAI / Claude / local LLM** — BookDate recommendation generation

These integrations are configured via the app's Settings pages (Plex connection + token, Prowlarr URL + API key, download client credentials, AI provider key, OIDC provider, notification webhooks for Discord/Pushover) and are not part of ReadMeABook's own exposed surface.

## Error Handling

No documented error response schema is available for the internal API. For the one endpoint this deployment actually depends on (`/api/health`), treat it per standard HTTP semantics: 2xx = healthy, anything else (4xx/5xx/connection refused/timeout) = unhealthy, which is exactly how the liveness/readiness probes interpret it.

## Summary

| Capability | Status in this deployment |
|---|---|
| Public documented REST API | Not provided by upstream |
| Health endpoint | `GET /api/health` on port 3030 — used for k8s probes |
| API keys for integrations | Not configured/observed |
| Webhooks (outbound) | Not configured in this deployment |
| SDKs | None published upstream |
| OIDC/OAuth | Supported by the app, not verified as configured here |

---

*This documentation reflects what was observed in the `apps/media/readmeabook/helmrelease.yml` manifest and live cluster state for this homelab deployment. It is not a reproduction of upstream API documentation, because the upstream project does not publish a stable public API.*
