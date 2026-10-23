---
title: Homepage User Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [user-guide, documentation, dashboard, homelab]
---

# Homepage User Guide

## Introduction

Homepage is the landing page for the Conder Home homelab — it's what you see when you go to `https://homepage.conderhome.xyz`. It links out to every other self-hosted service (Plex, Sonarr, Grafana, Pi-hole, Proxmox, Vaultwarden, and more) and shows a live-status "widget" on many of them so you can tell at a glance whether something is up, busy, or needs attention. This guide covers using the dashboard day-to-day; see the Deployment Guide for how to add or change what's on it.

## Getting Started

### Accessing the Dashboard

1. Browse to `https://homepage.conderhome.xyz` (accessible from the homelab network/VPN)
2. The page loads with a background image, a title ("Conder Home Homelab"), and a row of info widgets at the top
3. Below that, service tiles are organized into four groups: **Media**, **Network**, **Services**, **Infrastructure**

### Dashboard Overview

When you load Homepage you'll see:

- **Info widgets row**: local resource usage (CPU/memory/disk), a search bar, and current weather
- **Service groups**: tiled links to every homelab app, grouped by category
- **Live status on many tiles**: a small widget area showing real-time data pulled from that service's own API (e.g. active Plex streams, Sonarr/Radarr queue size, Pi-hole blocked-query stats)
- **Status ping dot**: on services with a `ping` configured, a colored dot shows reachability independent of any widget

## Using the Dashboard

### Service Groups

| Group | What's in it |
|-------|--------------|
| Media | Plex, TrueNAS, ReadMeABook, Radarr, Sonarr, Prowlarr, qBittorrent |
| Network | Unifi, MySpeed, Pi-hole (x2), Omada/Wireless, Guest Network, KeyShift |
| Services | Termix, NGINX Proxy Manager, Wireguard, Actual Budget, Vaultwarden, Immich, NexusTale |
| Infrastructure | Authentik, Proxmox, Dockhand, Grafana, Docs, Hermes, HomeAssistant |

### Clicking a Tile

Click anywhere on a tile (not on the widget data itself) to open that service's `href` link — it opens the service's actual web UI, taking you away from Homepage. Widgets never require you to log in separately on the dashboard; Homepage fetches the live data server-side using stored credentials, so you don't need to authenticate again just to see the summary.

### Reading Widget Data

Widget content varies per service type:

- **Plex**: current stream count, movie/TV library totals
- **Radarr/Sonarr/Prowlarr**: queue size and indexer status
- **qBittorrent**: leech progress/size (if enabled)
- **Pi-hole**: total queries, blocked count, blocked percentage, gravity list size
- **TrueNAS**: pool status/usage
- **Immich**: photo/video library stats
- **Grafana**: active alert count
- **Home Assistant**: selected sensor values (temperature, daily energy use, number of switches currently on, wind speed) — these are custom-picked entities, not a generic HA summary
- **Hermes**: health status and running version, pulled from its `/api/health` endpoint

### Info Widgets (top row)

- **Resources**: CPU/memory usage and disk usage (`/`) of the node the Homepage pod is running on — this reflects that one node, not the whole cluster
- **Search**: a DuckDuckGo search box, opens results in a new tab
- **Weather**: current conditions for a fixed location, shown in imperial units

### Bookmarks

Not visible on the main service grid by default in most Homepage themes — bookmarks (GitHub, Reddit, YouTube under their own groups) appear in a separate bookmarks panel/sidebar depending on theme settings. They're plain links with no widgets or status.

## Settings You Can See But Shouldn't Edit From the Browser

Homepage has no in-browser settings/admin panel for this deployment — there is nothing to click to "configure" the dashboard from the UI itself. All appearance and content changes are made by editing YAML files in the GitOps repo (see Deployment Guide). If you want the title changed, a new service added, or the background swapped, that's a `git` change, not a dashboard click.

## Troubleshooting

### Common Issues

**Issue**: A tile shows no live data (blank widget area)
**Cause/Fix**: The target service's API is unreachable from inside the cluster, or its credentials/API key in `services.yaml` are stale (e.g. after rotating an API key in Sonarr/Radarr/Pi-hole, the corresponding `key:` in `services.yaml` must be updated too).

**Issue**: Ping dot is red/offline but the service works fine when you visit it directly
**Cause/Fix**: The `ping` target address may differ from the public `href` (often an internal IP:port) and could be blocked by a firewall rule or the service may be bound to a different interface.

**Issue**: Whole dashboard won't load / shows an error page
**Cause/Fix**: A YAML syntax error was introduced in one of the config files. This needs a fix at the GitOps repo level and a Flux reconcile — see Deployment Guide → Troubleshooting.

**Issue**: Dashboard briefly unreachable
**Cause/Fix**: Homepage runs a single replica with a `Recreate` deploy strategy, so any image update or config-triggered redeploy causes a short gap while the old pod terminates and the new one starts. This is expected, not a fault.

**Issue**: New service I asked to be added isn't showing up yet
**Cause/Fix**: Config changes go through Git → Flux reconcile (up to the `interval: 1h`, or immediate if a push triggers Flux's notification webhook) → ConfigMap update → kubelet file sync. Allow a few minutes; a browser refresh picks up the new file.

### Getting Help

- **Live config reference**: upstream docs at https://gethomepage.dev/configs/
- **Repo location**: `apps/tools/homepage/` in the `infra-k8s` GitOps repo
- **Logs**: `kubectl logs -n tools -l app.kubernetes.io/name=homepage`
- **Pod status**: `kubectl get pods -n tools -l app.kubernetes.io/name=homepage`

## Tips

- Use the ping dots as a first-glance health overview before diving into Grafana.
- If a widget looks wrong after rotating a credential/API key in the underlying service, that's almost always the fix — update the matching field in `services.yaml`.
- Group placement in `services.yaml` directly mirrors the on-screen grouping, top to bottom as written in the file.

---

*This guide covers day-to-day use of the Homepage dashboard in the Conder Home homelab. For adding/changing services, see the Deployment Guide.*
