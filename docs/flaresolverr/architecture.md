---
title: FlareSolverr Architecture Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [architecture, documentation, flaresolverr, kubernetes, proxy]
---

# FlareSolverr Architecture Guide

## Overview

FlareSolverr is a headless, browser-automation proxy service that solves Cloudflare "I'm Under Attack" challenges, CAPTCHAs, and other bot-detection JavaScript challenges on behalf of other services. It has **no user-facing UI and no end-user workflow** — it exists purely as an internal dependency consumed by Prowlarr (and, by extension, Sonarr/Radarr indexer searches that route through Prowlarr) when an indexer is fronted by Cloudflare protection that a plain HTTP client cannot pass.

In this cluster, FlareSolverr runs as a single-replica Deployment in the `media` namespace, exposed only via an internal ClusterIP Service on port `8191`. It is never exposed through an Ingress and has no TLS/hostname of its own — it is reachable only from other pods inside the cluster (primarily Prowlarr).

## System Architecture

### High-Level Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                      media namespace (k3s)                    │
│                                                                 │
│  ┌───────────────┐   HTTP POST /v1    ┌─────────────────────┐ │
│  │   Prowlarr     │ ─────────────────▶ │   FlareSolverr Pod   │ │
│  │  (indexer      │   JSON request     │                      │ │
│  │   manager)     │ ◀───────────────── │  ┌────────────────┐ │ │
│  └───────────────┘   JSON response     │  │  Flask HTTP API │ │ │
│         │                              │  └────────────────┘ │ │
│         │                              │  ┌────────────────┐ │ │
│         │  (configured as a            │  │ Selenium /     │ │ │
│         │   FlareSolverr proxy         │  │ undetected-     │ │ │
│         │   on specific indexers)      │  │ chromedriver   │ │ │
│         │                              │  └────────────────┘ │ │
│         ▼                              │  ┌────────────────┐ │ │
│  External indexer site                 │  │ Headless        │ │ │
│  (Cloudflare-protected)  ◀──────────── │  │ Chromium        │ │ │
│                           outbound     │  └────────────────┘ │ │
│                           browser req. └─────────────────────┘ │
└───────────────────────────────────────────────────────────────┘
```

### Core Components

#### 1. Flask HTTP API Layer
- Single internal endpoint: `POST /v1`
- Accepts a JSON "command" body (`request.get`, `request.post`, `sessions.create`, `sessions.destroy`, `sessions.list`)
- Synchronous request/response — the caller blocks while the challenge is solved
- No authentication, no TLS — it relies entirely on network isolation (ClusterIP only, no Ingress)

#### 2. Browser Automation Engine
- **undetected-chromedriver / Selenium**: drives a real headless Chromium instance patched to evade basic bot-fingerprinting
- Loads the target URL, waits for the Cloudflare JS challenge to resolve, and extracts the resulting cookies (`cf_clearance`) and page HTML
- Returns the solved cookies and response body to the caller so it can replay the request itself, or returns the fully rendered page directly

#### 3. Session Management
- Optional persistent "sessions" keep a browser context (and its cookies) alive across multiple calls, avoiding re-solving the challenge on every request
- Sessions are in-memory only — a pod restart drops all sessions and any indexer using a persistent session will need to re-establish it

#### 4. Single Container, No External Dependencies
- No database, no cache, no message queue — the entire service is one stateless (aside from in-memory sessions) container
- `CAPTCHA_SOLVER` is configured as `none` in this deployment, meaning it will solve Cloudflare's JS challenge but will **not** attempt to solve hCaptcha/reCAPTCHA — those require a separately configured third-party solver, which is not enabled here

### Data Flow Architecture

#### Indexer Search / Grab Flow

```
1. User/Sonarr/Radarr triggers search in Prowlarr
          ↓
2. Prowlarr determines indexer X requires FlareSolverr
          ↓
3. Prowlarr POSTs { cmd: "request.get", url: "<indexer-url>" } to
   http://flaresolverr.media.svc.cluster.local:8191/v1
          ↓
4. FlareSolverr launches/reuses a headless Chromium session
          ↓
5. Chromium loads the page, Cloudflare's JS challenge executes,
   cf_clearance cookie is issued
          ↓
6. FlareSolverr returns { status: "ok", solution: { url, status,
   cookies, response, userAgent } } to Prowlarr
          ↓
7. Prowlarr uses the returned cookies/response to complete the
   indexer query and parse search results
```

If step 3's challenge-solve exceeds FlareSolverr's internal timeout (default 60s, configurable via Prowlarr's indexer proxy settings), FlareSolverr returns an error and the indexer search in Prowlarr fails for that indexer only — other indexers are unaffected.

### Deployment Architecture

#### Containerization
- **Single upstream image**: `ghcr.io/flaresolverr/flaresolverr:latest` (no custom build in this repo)
- Bundles Chromium + chromedriver + Python Flask app in one image
- No init containers, no sidecars

#### Orchestration (Flux / Helm)
- Deployed via a Flux `HelmRelease` using the shared `bjw-s/app-template` chart (same pattern as every other app in this cluster)
- `spec.chart.spec.version: "3.x"` tracks the app-template major version; FlareSolverr's own app version is pinned to `latest` on the image tag (no image automation/tag policy configured for this app, unlike Prowlarr)
- Single controller named `flaresolverr`, one container named `app`

#### Resource Management
- Requests: `100m` CPU / `256Mi` memory
- Limits: `1000m` CPU / `1024Mi` memory
- Headless Chromium is the dominant resource consumer — CPU spikes during challenge-solving (page load + JS execution), memory climbs with concurrent/persistent sessions
- No horizontal scaling — Prowlarr is configured with a single FlareSolverr proxy URL, and FlareSolverr itself does not support clustering/shared session state across replicas, so this stays a single replica

#### Networking
- `service.flaresolverr`: ClusterIP Service, port `8191`, selector matches the `flaresolverr` controller
- **No Ingress** — intentionally not exposed outside the cluster; there is nothing for an external client to usefully call
- **No PersistentVolume** — fully stateless; losing the pod loses in-memory sessions only, nothing on disk needs to survive

### Security Architecture

- No authentication on the `/v1` endpoint — security boundary is network-level only (ClusterIP confines access to in-cluster callers; NetworkPolicies, if present in the cluster, would further restrict this to the `media` namespace)
- Runs as the image's default user (no `securityContext` overrides defined in the HelmRelease)
- `automountServiceAccountToken: true` with the default ServiceAccount — FlareSolverr itself calls no Kubernetes API, so this token is unused by the application but inherited from the chart's defaults
- Outbound-only risk profile: the main exposure is that FlareSolverr's Chromium instance makes outbound requests to arbitrary indexer URLs supplied by Prowlarr — it should only ever be pointed at trusted indexer domains configured in Prowlarr

### Monitoring & Observability

- `LOG_LEVEL: info`, `LOG_FILE: none` (logs to stdout only, captured via standard pod logs), `LOG_HTML: "false"` (does not dump full page HTML on failures, keeping log volume manageable)
- No `/health` endpoint is wired into probes in this deployment — there are no liveness/readiness probes configured; pod health is inferred purely from container running state
- `kubectl logs -n media -l app.kubernetes.io/name=flaresolverr` is the primary way to see challenge-solve attempts and failures

## Technology Stack

- **Python** (Flask) — HTTP API layer
- **Selenium + undetected-chromedriver** — browser automation and anti-fingerprinting
- **Headless Chromium** — the actual browser that executes Cloudflare's JS challenge
- **Docker** — single multi-stage image published by the upstream FlareSolverr project
- **Flux CD + Helm (`bjw-s/app-template`)** — GitOps deployment, consistent with every other `apps/media/*` workload in this cluster

## Integration Points

- **Prowlarr** (`apps/media/prowlarr`): the sole internal consumer. Configured inside Prowlarr's own Indexer Proxies settings as a FlareSolverr-type proxy pointing at `http://flaresolverr.media.svc.cluster.local:8191`, then assigned to the specific indexers that require it.
- No other app in the cluster talks to FlareSolverr directly.

This architecture is intentionally minimal: one stateless container, one internal port, one consumer. There is no "end user" architecture to document because none exists — all value is delivered transitively through Prowlarr's indexer results.
