---
title: Radarr User Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [user-guide, documentation, movies, radarr]
---

# Radarr User Guide

## Introduction

Radarr automates finding, downloading, and organizing movies. You tell it which movies you want (monitored), it searches configured indexers for matching releases, sends the best one to your download client, and files the finished result into your movie library with consistent naming. This guide covers day-to-day usage of the instance running at `https://radarr.conderhome.xyz`.

## Getting Started

### First Time Setup

1. **Access the Web Interface** at `https://radarr.conderhome.xyz`
2. **Log in** using the forms-authentication credentials configured for this instance
3. **Verify Root Folder**: Settings → Media Management → Root Folders should list `/movies`
4. **Verify Indexers**: Settings → Indexers should show indexers synced in from Prowlarr (configured separately in Prowlarr, not here)
5. **Verify Download Client**: Settings → Download Clients should list your configured client (e.g. qBittorrent/SABnzbd)

### Dashboard Overview

When you first access Radarr, you'll see:

- **Movies**: your full library grid/list, with monitoring and availability status
- **Calendar**: upcoming/recent releases for movies you're tracking
- **Activity → Queue**: downloads in progress or awaiting import
- **Activity → History**: completed grabs, imports, and failures
- **Wanted → Missing**: monitored movies without a file yet

## Managing Your Movie Library

### Adding Movies

1. Click **"Add New"** (or the `+` icon)
2. Search by title — results come from Radarr's metadata provider (TMDb-backed)
3. Select the correct match (check year/poster to avoid mismatches with remakes)
4. Choose:
   - **Root Folder**: `/movies`
   - **Quality Profile**: controls which releases are acceptable/preferred
   - **Monitored**: on, to let Radarr search/grab automatically
   - **Minimum Availability**: `Announced`, `In Cinemas`, or `Released` — controls how early Radarr will start searching
5. Click **"Add Movie"** — Radarr immediately performs a search if "Start search on add" is enabled

### Searching for Releases

#### Automatic Search

Radarr periodically runs RSS sync against your indexers and automatically grabs matching releases for monitored, missing movies — no action needed once a movie is added and monitored.

#### Manual Search

For more control, or when automatic search hasn't found anything acceptable:

1. Open the movie's detail page
2. Click **"Search"** (manual interactive search)
3. Review the release list — quality, size, seeders/age, and indexer are shown
4. Click **"Download"** on the release you want

### Managing the Queue

- **Activity → Queue** shows active downloads with progress and ETA
- Items stuck for too long can be removed, optionally blocklisting the release so it isn't re-grabbed
- Import failures (e.g. unsupported file layout, missing video file) are surfaced here with a reason

### Organizing Your Library

#### Renaming

Radarr renames imported files according to a configurable naming format (Settings → Media Management → Movie Naming). Common tokens: `{Movie Title}`, `{Release Year}`, `{Quality Full}`.

#### Monitoring Status

Each movie has a monitored/unmonitored toggle:

- **Monitored**: Radarr will search for and auto-grab new releases, and alert on upgrades
- **Unmonitored**: kept in the library for reference but Radarr won't search for it

#### Quality Profiles

Quality profiles define which qualities are acceptable and which is the target "cutoff" for upgrades:

| Setting | Purpose |
|---------|---------|
| Allowed Qualities | Which quality tiers Radarr will grab at all |
| Upgrade Allowed | Whether Radarr keeps upgrading to better qualities after the first grab |
| Cutoff | The quality at which Radarr stops upgrading |

### Removing Movies

1. Open the movie detail page → **"Delete Movie"**
2. Optionally check **"Delete Files"** to remove the file(s) from `/movies`
3. Optionally check **"Add List Exclusion"** to prevent it from being automatically re-added by list sync, if list-based imports are in use

## Indexers and Search (via Prowlarr)

Radarr's indexer list in this deployment is managed by **Prowlarr**, not configured directly in Radarr:

- Prowlarr pushes its configured indexers into Radarr using Radarr's API
- Indexers behind Cloudflare/CAPTCHA protections are configured in Prowlarr to proxy through **FlareSolverr**
- If searches are failing for specific indexers, check Prowlarr first (indexer status, FlareSolverr connectivity), then check Radarr's Settings → Indexers to confirm the sync landed correctly

## Settings and Configuration

### General Settings

- **Port**: `7878` (internal; exposed externally via ingress on 443)
- **URL Base**: empty (served from root path)
- **Authentication**: Forms-based login, required
- **API Key**: Settings → General → Security (also readable from `/config/config.xml` on the config PVC)

### Media Management

- **Root Folders**: `/movies`
- **Rename Movies**: enable/disable automatic renaming on import
- **File Management**: recycling bin location, permissions, hardlink/copy behavior

### Download Client Settings

- Configure the external download client (qBittorrent/SABnzbd/etc.) with its own host/port/category
- Radarr only talks to the download client's API — it does not download files itself

### Notification (Connect) Settings

Configure outbound notifications (Discord, Slack, webhook, etc.) under Settings → Connect to get alerts for grabs, imports, upgrades, and health issues.

## Advanced Features

### Lists

Radarr supports automatically importing movies from external lists (IMDb lists, TMDb lists, Trakt, etc.) under Settings → Lists — not configured by default in this deployment but available if you want hands-off collection building.

### Custom Formats

Custom Formats let you score releases based on naming patterns (e.g. preferred release groups, HDR formats, disliked terms) layered on top of quality profiles for finer-grained release selection.

### Backups

Settings → General → Backups lets you trigger or schedule backups of the database and config to `/config/Backups`. Download a backup before major changes (quality profile overhauls, naming format changes).

## Troubleshooting

### Common Issues

#### No Search Results

**Issue**: Manual/automatic search returns nothing
**Solution**: Check Settings → Indexers for errors; confirm Prowlarr sync is active; for Cloudflare-protected indexers, confirm FlareSolverr is reachable from Prowlarr

#### Download Stuck in Queue

**Issue**: Item sits in queue without progressing
**Solution**: Check the download client directly (is it running, is the torrent/NZB actually downloading); remove and blocklist if the release is dead

#### Import Failing After Download Completes

**Issue**: File downloads but Radarr won't import it
**Solution**: Check Activity → Queue for the failure reason — common causes are unsupported file structure, no matching video file found, or permission issues on `/downloads` or `/movies`

#### Wrong Movie Matched

**Issue**: Radarr grabbed/matched the wrong movie (common with remakes/sequels)
**Solution**: Use "Unmonitored" + manual re-match from the movie's detail page, or delete and re-add with the correct TMDb match

### Getting Help

- **Logs**: Settings → System → Logs in the UI, or `kubectl logs -n media deploy/radarr`
- **Health page**: System → Status → Health surfaces indexer/download client/root folder issues directly
- **Community**: Servarr wiki and forums for general Radarr troubleshooting

## Best Practices

### Library Organization

- Keep a consistent naming format so files are predictable for other tools consuming `/movies`
- Review the "Unmonitored" list periodically to avoid wasted searches on movies you no longer want
- Use quality profile cutoffs deliberately — "always upgrade" profiles can cause repeated re-downloads

### Reliability

- Keep FlareSolverr and Prowlarr healthy — a surprising number of "no results" issues trace back to a broken Prowlarr → indexer or Prowlarr → FlareSolverr link, not Radarr itself
- Take a backup before bulk-editing quality profiles or naming formats

### Security

- Treat the API key like a password — anything with it can add/remove movies and trigger searches
- Keep forms authentication enabled; don't disable it even behind the internal ingress

---

*This user guide covers the Radarr deployment at `radarr.conderhome.xyz` in this cluster. For deeper Radarr behavior details, see the architecture and API documentation alongside this guide.*
