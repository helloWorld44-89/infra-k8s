---
title: Homepage API Documentation
date: 2026-10-23
author: Hermes Docs Agent
tags: [api, documentation, dashboard, widgets, yaml]
---

# Homepage API Documentation

## Overview

Homepage does not expose a conventional REST API for consumers to call — it is a dashboard, not a managed service. Its "API" is its **YAML configuration schema**: every piece of behavior (what tiles appear, what live data they show, how the dashboard looks) is declared in the seven config files mounted into the container. This document describes that configuration API: the schema of `services.yaml` (the primary file edited in this cluster), the widget types in use, and the few HTTP endpoints Homepage itself serves or calls.

**Config root** (in-container): `/app/config/`
**Config root** (in-repo): `apps/tools/homepage/configmaps/`
**Dashboard URL**: `https://homepage.conderhome.xyz`

## Configuration Files Reference

| File | Schema docs (upstream) | Used for |
|------|------------------------|----------|
| `services.yaml` | https://gethomepage.dev/configs/services/ | Groups, service tiles, links, widgets |
| `settings.yaml` | https://gethomepage.dev/configs/settings/ | Title, background, layout, providers |
| `widgets.yaml` | https://gethomepage.dev/configs/info-widgets/ | Info widgets (resources, search, weather) |
| `bookmarks.yaml` | https://gethomepage.dev/configs/bookmarks | Bookmark-only links |
| `docker.yaml` | https://gethomepage.dev/configs/docker/ | Docker socket/host integration (unused here) |
| `kubernetes.yaml` | (native k8s widget config) | Native k8s cluster widget (unused here, placeholder) |
| `proxmox.yaml` | (Proxmox cluster widget config) | Cluster-wide Proxmox widget (unused here, placeholder) |

## `services.yaml` Schema

Top-level structure is a YAML list of **groups**, each a single-key mapping from group name to a list of **service** entries:

```yaml
- <Group Name>:
    - <Service Name>:
        icon: <icon-name-or-url>
        href: <link-url>
        description: <optional text>
        ping: <optional-url-or-ip-to-health-check>
        widget:
          type: <widget-type>
          url: <widget-backend-url>
          # ...widget-type-specific fields
```

### Field Reference (per service entry)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `icon` | string | no | Icon filename from Homepage's bundled icon set (e.g. `plex.png`), a [Simple Icons](https://simpleicons.org) slug (`si-nousresearch.png`), or a full icon URL |
| `href` | string | no | Click-through link for the tile |
| `description` | string | no | One-line subtitle shown under the service name |
| `ping` | string (URL/host) | no | Address Homepage pings to show an online/offline indicator dot, independent of any widget |
| `widget` | object | no | Live-status widget block (see below) |

Groups in the live config, in order: **Media**, **Network**, **Services**, **Infrastructure**.

## Widget Types in Use

### Native integration widgets

Each of these maps `type:` to a built-in adapter that knows the target app's API. Common required fields are `type` and `url`; most also need `key` (API key) or `username`/`password`.

| `type` | Example service | Auth fields | Notable options |
|--------|-----------------|-------------|------------------|
| `plex` | Plex | `key` | `fields: ["streams","movies","tv"]` |
| `truenas` | TrueNAS | `username`, `password` | `enablePools: true`, `nastype: scale` |
| `radarr` | Radarr | `key` | `enableQueue: true` |
| `sonarr` | Sonarr | `key` | `enableQueue: true` |
| `prowlarr` | Prowlarr | `key` | — |
| `qbittorrent` | qBittorrent | `username`, `password` | `enableLeechProgress`, `enableLeechSize` |
| `unifi` | Unifi Controller | `username`, `password` (or `key`) | — |
| `myspeed` | MySpeed | none | — |
| `pihole` | Pi-hole (x2 instances) | `key` | `version: 6`, `fields: ["queries","blocked","blocked_percent","gravity"]` |
| `omada` | Omada Controller | `username`, `password` | `site: <site-name>` |
| `npm` | NGINX Proxy Manager | `username`, `password` | — |
| `immich` | Immich | `key` | `version: 2` |
| `authentik` | Authentik | `key` | currently commented out in config |
| `proxmox` | Proxmox (per-host) | `username` (`user@realm!tokenid`), `password` (token secret) | `node:` optional |
| `dockhand` | Dockhand | `username`, `password` | required for local auth |
| `grafana` | Grafana | `username`, `password` | `version: 2`, `alerts: grafana` — uses basic-auth embed against Grafana's API, not SSO |
| `homeassistant` | Home Assistant | `key` (long-lived access token) | `custom:` array of state/template entries (see below) |

### `customapi` widget (generic JSON adapter)

Used when there's no dedicated Homepage integration for a service — here, for the in-house **Hermes** tile:

```yaml
- Hermes:
    icon: si-nousresearch.png
    href: http://192.168.5.99:9119
    widget:
      type: customapi
      url: http://192.168.5.99:9119/api/health
      mappings:
        - field: ok
          label: Status
          format: text
        - field: displayVersion
          label: Version
          format: text
```

- `url`: any JSON HTTP(S) endpoint Homepage can reach server-side
- `mappings`: array of `{ field, label, format }` — `field` is the JSON key (dot-path for nested fields) read from the response, `label` is the display name, `format` controls rendering (`text`, `number`, `percent`, `bytes`, etc.)
- This is the pattern to copy for any future internal/custom service with a simple JSON status endpoint.

### `homeassistant` `custom` entries

Home Assistant's widget supports arbitrary sensor pulls beyond the built-in fields:

```yaml
custom:
  - state: sensor.mts300_temperature
  - state: sensor.total_energy_today
    label: energy today
  - template: "{{ states.switch|selectattr('state','equalto','on')|list|length }}"
    label: switches on
  - state: weather.forecast_home
    label: wind speed
    value: "{attributes.wind_speed} {attributes.wind_speed_unit}"
```

- `state`: entity ID to read `.state` from
- `template`: a Home Assistant Jinja2 template evaluated server-side
- `value`: a Python-style format string pulling from the entity's `attributes`
- `label`: optional override of the displayed label (defaults to the entity's friendly name)

## `settings.yaml` Schema (as deployed)

```yaml
title: Conder Home Homelab
background: <unsplash-image-url>
cardBlur: md
layout:
  media:
    style: row
    column: 4
providers:
  openweathermap: <api-key>
  weatherapi: <api-key>
```

| Field | Description |
|-------|--------------|
| `title` | Browser tab / dashboard title |
| `background` | Background image URL |
| `cardBlur` | Backdrop blur level for cards (`sm`, `md`, etc. — Tailwind blur scale) |
| `layout.<group>.style` / `.column` | Per-group layout overrides (here, the `Media` group renders as a 4-column row) |
| `providers.*` | API keys for weather providers consumed by `widgets.yaml`'s `openmeteo`/weather entries |

## `widgets.yaml` Schema (as deployed)

```yaml
- resources:
    cpu: true
    memory: true
    disk: /
- search:
    provider: duckduckgo
    target: _blank
- openmeteo:
    label: Home
    latitude: 35.5965
    longitude: -88.6314
    timezone: America/Chicago
    units: imperial
    cache: 5
    format:
      maximumFractionDigits: 1
```

These render above the service groups: a local resource monitor (CPU/memory/disk of the Homepage pod's node), a search bar, and a weather widget for a fixed lat/long.

## `bookmarks.yaml` Schema (as deployed)

```yaml
- <Group>:
    - <Site Name>:
        - abbr: <2-letter-badge>
          href: <url>
```

Simple link list (GitHub, Reddit, YouTube), rendered separately from the service groups — no widgets, no health pings.

## HTTP Surface Homepage Itself Serves

Homepage is primarily consumed through its single rendered page; it does not expose a documented public REST API for third parties in this deployment. Relevant endpoints:

| Endpoint | Purpose |
|----------|---------|
| `GET /` | Main dashboard page |
| `GET /api/*` (internal Next.js API routes) | Server-side routes the frontend uses to fetch widget data; not intended for external consumption and unversioned/undocumented upstream |

For scripted health checks against Homepage itself, use the `ping`/`href` target (`GET /`) rather than any internal API route.

## Error Handling (widget-level)

Homepage does not raise cluster-visible errors for a bad widget config — failures degrade gracefully in the UI:

| Symptom on dashboard | Likely cause |
|----------------------|--------------|
| Tile renders with no live data / dashes | Widget `url` unreachable, DNS failure, or TLS error from inside the pod |
| "Missing or invalid credentials" style text on tile | Wrong `key`/`username`/`password` for that widget type |
| Tile shows but `ping` dot never goes green | `ping` target not reachable from inside the cluster network (e.g. host firewalled) |
| Whole dashboard fails to load | Malformed YAML in `services.yaml`/`settings.yaml` — check pod logs (`kubectl logs`) for a YAML parse error |

## Adding a Custom Integration

For any internal service that lacks a built-in Homepage widget type, expose a small JSON endpoint and wire it up with `customapi` (see the Hermes example above) — this requires no code change to Homepage itself, only a `services.yaml` edit (see the Deployment Guide for the full add-a-service procedure).

---

*This documentation describes Homepage's configuration-as-API surface as deployed in the Conder Home homelab. For the full upstream schema and every supported widget type, see https://gethomepage.dev/configs/.*
