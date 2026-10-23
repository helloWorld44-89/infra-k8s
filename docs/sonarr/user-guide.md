---
title: Sonarr User Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [user-guide, documentation, tv, media-automation]
---

# Sonarr User Guide

## Introduction

Sonarr is a self-hosted TV show collection manager: it tracks series you want, monitors indexers (via Prowlarr) for new or better-quality episode releases, sends them to your download client, and files the finished episodes into your TV library with clean, consistent naming. This guide covers day-to-day use at `https://sonarr.conderhome.xyz`.

## Getting Started

### First Time Setup

1. **Access the Web Interface** at `https://sonarr.conderhome.xyz`
2. **Set a Root Folder**: Settings → Media Management → Root Folders → add `/tv/TV Shows`
3. **Create/Verify a Quality Profile**: Settings → Profiles — defines which qualities to grab and which to upgrade toward
4. **Confirm Indexers Are Present**: Settings → Indexers — these should already be populated by Prowlarr's application sync; if the list is empty, check the Prowlarr integration first
5. **Add a Download Client**: Settings → Download Clients → point at your SABnzbd/qBittorrent/etc. instance
6. **Add Your First Series** and let Sonarr do the rest

### Dashboard Overview

When you open Sonarr, you'll see:

- **Series Library**: Poster/list view of everything you track
- **Calendar**: Upcoming and recently aired episodes
- **Activity Queue**: Downloads currently in progress
- **Wanted (Missing)**: Episodes you're monitoring but don't have yet

## Managing Your TV Library

### Adding a Series

1. Click **"Add New"** → **"Add New Series"**
2. Search by show title (Sonarr queries TheTVDB via SkyHook)
3. Select the correct match (check year/network to avoid duplicates/remakes)
4. Choose:
   - **Root Folder**: `/tv/TV Shows`
   - **Quality Profile**: e.g., HD-1080p
   - **Series Type**: Standard, Anime, or Daily (for talk shows/news with date-based episodes)
   - **Monitor**: All Episodes, Future Only, Missing Only, or None
5. Optionally check **"Start search for missing episodes"** to immediately search for everything monitored
6. Click **"Add Series"**

### Monitoring Options

Sonarr lets you fine-tune exactly what gets downloaded:

- **Series-level monitoring**: Toggle the whole show on/off from automatic search/grab
- **Season-level monitoring**: Click a season to monitor/unmonitor just that season (useful to skip a season you already have from another source)
- **Episode-level monitoring**: Click into a season to toggle individual episodes

### Searching for Episodes

#### Automatic (RSS Sync)

Sonarr polls indexers (through Prowlarr) on a schedule (Settings → Indexers → RSS Sync Interval) and automatically grabs matching releases for anything monitored.

#### Manual Search

1. Open a series → select a season or episode
2. Click the **search icon** (magnifying glass)
3. Review candidate releases — quality, size, seeders/age, indexer
4. Click **"Download"** on the preferred release (manual/interactive search)

#### Searching for a Whole Series

From the series page, use the **"Search Monitored"** action (or trigger `SeriesSearch` via the API) to search all monitored, missing episodes at once.

### Understanding the Activity Queue

The Queue tab shows:

- **Title**: Parsed release name
- **Status**: Downloading, Importing, Warning, Failed
- **Progress/Size**: Download progress and total size
- **Indexer / Download Client**: Where it came from and where it's downloading

Failed or stuck items can be removed, optionally with "blocklist" to prevent Sonarr from re-grabbing the same release.

### Quality Profiles & Upgrades

| Setting | Purpose |
|---------|---------|
| Allowed Qualities | Which quality tiers Sonarr will grab at all |
| Upgrade Allowed | Whether Sonarr keeps searching for a better version after the first grab |
| Quality Cutoff | The quality level at which Sonarf stops upgrading further |
| Minimum/Preferred/Maximum Custom Format Score | Fine-grained scoring for release groups, codecs, etc. |

Once an episode file meets or exceeds the cutoff, Sonarr stops considering it for upgrade searches.

### Renaming & File Organization

Settings → Media Management lets you control:

- **Folder structure**: Season folders on/off, naming tokens
- **Episode naming format**: e.g., `{Series Title} - S{season:00}E{episode:00} - {Episode Title} {Quality Full}`
- **Rename existing files**: Series → "..." menu → Rename — applies the current naming format retroactively

## Settings and Configuration

### General Settings

- **Port / URL Base**: Internal service port is `8989`; exposed externally only via the Traefik ingress
- **Authentication**: Forms/Basic auth can be enabled (Settings → General → Security) — currently relies on the cluster's network boundary and TLS ingress
- **API Key**: Shown here; required for any REST API or Prowlarr/automation integration

### Media Management

- **Root Folders**: `/tv/TV Shows` (backed by the `tv-pvc` claim)
- **Episode Naming**: Customize per-episode file naming
- **File Management**: Use hardlinks when possible (keeps downloads and library in sync without double disk usage, since `/downloads` and `/tv/TV Shows` can be on the same underlying storage)
- **Unmonitor Deleted Episodes**: Prevents re-grabbing episodes you manually removed

### Indexers (managed via Prowlarr)

Indexers in this deployment are **not** meant to be added directly in Sonarr's Settings → Indexers page. Instead:

1. Add/maintain indexers centrally in Prowlarr (`prowlarr.conderhome.xyz`)
2. In Prowlarr, go to Settings → Apps → add Sonarr as an application (using Sonarr's URL + API key)
3. Prowlarr syncs its indexer list into Sonarr automatically

If an indexer is missing in Sonarr, check the Prowlarr → Apps sync status first rather than adding it manually (manual additions get out of sync on the next Prowlarr sync).

### Download Clients

Add your download client (SABnzbd, NZBGet, qBittorrent, Deluge, etc.) under Settings → Download Clients, pointing at its in-cluster service address and API key/credentials. Sonarr and the download client both need visibility into the shared `/downloads` path for import to work without network copies.

### Connect (Notifications)

Settings → Connect lets you wire up notifications/webhooks for events like Grab, Download/Import, Upgrade, Health Issue, and Series Delete — useful for Discord alerts or custom automation hooks.

## Troubleshooting

### Common Issues

#### No Indexers Showing Up

**Issue**: Settings → Indexers is empty, RSS sync has nothing to search
**Solution**: Check Prowlarr → Settings → Apps → Sonarr sync status; re-test/re-sync the application connection; verify the API key hasn't been regenerated on either side

#### Cloudflare-Protected Indexer Failing

**Issue**: A specific indexer repeatedly fails with challenge/CAPTCHA-like errors
**Solution**: This is handled by FlareSolverr, configured per-indexer in Prowlarr (not in Sonarr). Check the FlareSolverr pod is healthy (`kubectl get pods -n media -l app.kubernetes.io/name=flaresolverr`) and that the affected indexer in Prowlarr has FlareSolverr assigned as its proxy.

#### Downloads Not Importing

**Issue**: Episode finishes downloading but never shows up in the library
**Solution**: Check the download client's completed-download path matches a path Sonarr can also see (`/downloads`); check Activity → History for import failure reasons; verify file permissions on the shared PVC

#### Wrong Series Matched

**Issue**: Added the wrong show (e.g., a remake or regional variant)
**Solution**: Remove the series (optionally delete files), re-add and manually confirm the TVDB ID/year before confirming

#### Series Stuck "Missing" Despite Being Available

**Issue**: Monitored episode never gets grabbed
**Solution**: Run a manual interactive search on that episode to see why releases are being rejected (quality profile mismatch, custom format score, already in blocklist)

### Getting Help

- **Logs**: Settings → System → Logs, or `kubectl logs -n media deploy/sonarr`
- **Health Dashboard**: Settings → General banner / System → Status surfaces active health warnings
- **Community**: Sonarr's official Discord/forum and the Servarr wiki (wiki.servarr.com)

## Best Practices

### Library Organization

- Use season folders consistently; mixed flat/season-folder libraries complicate renaming
- Keep one quality profile per "tier" of show (e.g., Anime vs. Standard HD) rather than one-off per-series profiles
- Periodically run "Series → Refresh & Scan" after any out-of-band file changes

### Performance

- Keep the RSS sync interval reasonable (default ~15 min) — tightening it doesn't meaningfully speed up acquisition and adds indexer load
- Let Prowlarr own indexer health/backoff; don't duplicate indexers manually in Sonarr

### Reliability

- Because Sonarr runs as a single replica with `strategy: Recreate`, expect a short downtime window during pod restarts/updates — queued searches resume automatically once the pod is back
- Back up `/config` (the `sonarr-config` PVC) regularly; it holds the entire database, settings, and API key

---

*This user guide covers the Sonarr deployment at `sonarr.conderhome.xyz` as part of the homelab media automation stack (Sonarr, Radarr, Prowlarr, FlareSolverr). For authoritative upstream documentation, see the Servarr wiki.*
