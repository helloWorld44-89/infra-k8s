---
title: Prowlarr User Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [user-guide, documentation, media, indexers]
---

# Prowlarr User Guide

## Introduction

Prowlarr is the indexer manager for this homelab's media automation stack. Instead of adding and maintaining search indexers separately in Radarr and Sonarr, you configure them once in Prowlarr, and Prowlarr pushes that configuration out to both. Prowlarr also handles indexers that sit behind Cloudflare or basic CAPTCHA challenges by routing those requests through FlareSolverr, a companion service deployed alongside it.

## Getting Started

### Accessing Prowlarr

- **External**: `https://prowlarr.conderhome.xyz`
- **In-cluster**: `http://prowlarr.media.svc.cluster.local:9696`

### Dashboard Overview

The Prowlarr UI has four main sections:

- **Indexers**: Add, test, enable/disable individual trackers/indexer sites
- **Apps**: Connections to Radarr and Sonarr that receive synced indexer config
- **Settings**: Indexer proxies (FlareSolverr), general settings, API key, UI preferences
- **System**: Logs, tasks, health/status, updates

## Managing Indexers

### Adding an Indexer

1. Go to **Indexers → Add Indexer**
2. Search for or pick the tracker/indexer from the built-in catalog
3. Fill in required fields (base URL, API key, credentials — varies per indexer)
4. If the indexer is known to present a Cloudflare challenge, attach the **FlareSolverr** indexer proxy (see below)
5. Click **Test**, then **Save** once the test passes

### Testing and Troubleshooting Indexers

- **Test** button on each indexer runs a live connectivity/search check
- A failure that mentions an HTML challenge page, a timeout, or "Cloudflare" in the error message means the indexer needs a FlareSolverr proxy attached
- Indexer health is also visible under **System → Health**, which surfaces an `IndexerProxyCheck` warning if FlareSolverr itself becomes unreachable

### Using FlareSolverr for Protected Indexers

FlareSolverr is deployed as its own pod in the `media` namespace and is reachable only inside the cluster — it has no public URL and should not be given one.

1. Go to **Settings → Indexers → Indexer Proxies**
2. Click **Add Indexer Proxy** → choose **FlareSolverr**
3. Set **Host** to the in-cluster service address:
   ```
   http://flaresolverr.media.svc.cluster.local:8191
   ```
4. Set a reasonable **Request Timeout** (60s is a safe default — challenge-solving can take several seconds)
5. Click **Test** to confirm Prowlarr can reach FlareSolverr, then **Save**
6. On each indexer that needs it, open **Edit Indexer** and set **Indexer Proxy** to the FlareSolverr entry you just created

Once attached, every search/test request against that indexer is transparently routed through FlareSolverr first. You don't need to do anything differently when searching — Prowlarr handles the hand-off automatically.

### Removing or Disabling Indexers

- **Disable**: toggle the indexer off in the list — it stops being used for searches/sync but configuration is kept
- **Delete**: removes the indexer entirely, including its sync history; it will also be removed from Radarr/Sonarr on the next application sync

## Connecting Radarr and Sonarr

### Adding an Application

1. Go to **Settings → Apps → Add Application**
2. Choose **Radarr** or **Sonarr**
3. Set **Prowlarr Server** to Prowlarr's own in-cluster URL (`http://prowlarr.media.svc.cluster.local:9696`)
4. Set **Server** to the target app's in-cluster URL (e.g. `http://radarr.media.svc.cluster.local:7878`, `http://sonarr.media.svc.cluster.local:8989`)
5. Provide that app's API key (found in the target app's **Settings → General**)
6. **Test**, then **Save**

### How Syncing Works

- Prowlarr syncs automatically on an interval and whenever indexers are added, changed, or removed
- You can force an immediate sync via **System → Tasks → Application Indexer Sync**, or by running it on each app from the Apps list
- Indexer category tags determine routing: movie-category indexers sync to Radarr, TV-category indexers sync to Sonarr, indexers covering both sync to both
- **Important**: don't manually add/edit indexers directly inside Radarr or Sonarr — the next Prowlarr sync will overwrite those changes. Prowlarr is the source of truth.

## Settings and Configuration

### General Settings

- **API Key**: used by scripts/automation and by Radarr/Sonarr to reach back into Prowlarr if needed
- **Authentication**: can require a login for the UI (recommended since Prowlarr is exposed via Ingress at `prowlarr.conderhome.xyz`)
- **Log Level**: Trace/Debug/Info/Warn — raise to Debug/Trace when diagnosing a flaky indexer or FlareSolverr issue

### Indexer Proxy Settings

- **Host**: the FlareSolverr service URL — must be the in-cluster DNS name (`flaresolverr.media.svc.cluster.local:8191`), never a localhost or external address, since FlareSolverr has no Ingress
- **Request Timeout**: increase if FlareSolverr frequently times out solving challenges (slow challenges, cold-started headless browser sessions)
- **Tags**: optionally scope a proxy to only certain indexers via tags, if you have multiple proxy configurations

## Troubleshooting

### Common Issues

**Issue**: Indexer test fails with a timeout or raw HTML in the error
**Solution**: Attach a FlareSolverr indexer proxy to that indexer (see "Using FlareSolverr for Protected Indexers" above)

**Issue**: FlareSolverr proxy test fails ("connection refused" / "no route to host")
**Solution**: Confirm the FlareSolverr pod is running: `kubectl get pods -n media -l app.kubernetes.io/name=flaresolverr`. Confirm the Host field uses the exact in-cluster service address, not an external hostname.

**Issue**: Radarr/Sonarr don't show a newly added indexer
**Solution**: Check the indexer's category tags match what the application sync profile expects, then trigger a manual sync (**System → Tasks → Application Indexer Sync**)

**Issue**: Indexer searches are very slow when FlareSolverr is attached
**Solution**: This is expected — FlareSolverr launches/uses a real headless browser session per challenge. Only attach FlareSolverr to indexers that actually require it; don't attach it blanket to all indexers.

**Issue**: Prowlarr web UI is unreachable
**Solution**: Check pod status (`kubectl get pods -n media -l app.kubernetes.io/name=prowlarr`), then check the Traefik ingress/cert status for `prowlarr.conderhome.xyz`.

### Getting Help

- **Logs**: `kubectl logs -n media -l app.kubernetes.io/name=prowlarr`
- **FlareSolverr logs**: `kubectl logs -n media -l app.kubernetes.io/name=flaresolverr`
- **Upstream docs**: Servarr wiki (Prowlarr), FlareSolverr GitHub repo

## Best Practices

- Only attach FlareSolverr to indexers that genuinely need it — it adds real latency and resource cost (headless Chromium) per request
- Treat Prowlarr as the single source of truth for indexers; configure once, let it sync
- Periodically check **System → Health** for stale proxy or indexer warnings
- Keep the `prowlarr-config` PVC backed up — it holds indexer credentials and all configuration, with no external secret store backing it

---

*This user guide covers the Prowlarr deployment in the conderhome homelab k3s cluster.*
