---
title: Immich User Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [user-guide, documentation, photos, kubernetes]
---

# Immich User Guide

## Introduction

Immich is the self-hosted photo and video backup/management platform for this homelab, replacing the previous Docker-host deployment. This guide covers day-to-day use of the Web UI and mobile apps.

## Getting Started

### Accessing Immich

- **Web UI**: `https://immich.conderhome.xyz`
- **Mobile apps**: official Immich apps for iOS and Android, configured to point at `https://immich.conderhome.xyz` as the server URL
- **Login**: use your Immich account email/password (created by an admin, or self-registered if public registration is enabled)

### Dashboard Overview

After logging in, you'll see:

- **Timeline**: all photos/videos in chronological order (the main view)
- **Search bar**: natural-language smart search plus structured filters
- **Sidebar**: Albums, People, Places (map view), Favorites, Archive, Trash, Shared links
- **Upload button**: manual upload from the browser

## Managing Your Photo Library

### Uploading Photos and Videos

#### Method 1: Mobile App Auto-Backup (Recommended)

1. Install the Immich app and log in with your server URL and credentials
2. Enable **Background Backup** in app settings
3. Select which albums/folders on your device to back up
4. The app uploads automatically over Wi-Fi (or cellular, if enabled)

#### Method 2: Web Upload

1. Click the **Upload** button on the dashboard
2. Drag and drop files, or browse to select them
3. Upload progress is shown; assets appear in the timeline once processed

#### Method 3: CLI / Bulk Import

For large existing libraries, use the official Immich CLI to bulk-upload a directory tree (see the API documentation for CLI usage).

#### Supported Formats

- **Photos**: JPEG, PNG, HEIC/HEIF, DNG, and most common RAW formats
- **Videos**: MP4, MOV, and most common codecs (transcoded as needed for playback)

### Organizing Your Library

#### Albums

- Create albums manually and add photos from the timeline or during upload
- Albums can be **shared** with other Immich users or via a public link

#### People

- Immich automatically detects and clusters faces across your library (powered by the `immich-machine-learning` service)
- Name a detected person once, and all photos containing them are automatically tagged going forward

#### Places (Map View)

- Photos with GPS EXIF data appear on an interactive map
- Useful for browsing trips and location-based memories

#### Favorites and Archive

- **Favorite**: star photos you want to find quickly
- **Archive**: hide photos from the main timeline without deleting them (useful for screenshots, receipts, etc.)

### Searching Your Library

#### Smart Search

Type natural-language queries like "beach sunset" or "red car" — Immich uses CLIP embeddings (via `immich-machine-learning`) to find visually matching photos, not just ones with matching filenames or tags.

#### Structured Filters

Combine smart search with filters:

- **Date range**
- **Camera make/model**
- **City / location**
- **People** (specific recognized faces)
- **Media type** (photo vs. video)
- **Favorites / Archived**

## Sharing

### Shared Albums

1. Open an album → **Share** → invite another Immich user by email, or generate a **public link**
2. Public links can have an expiration date and optional password
3. Shared users can view (and optionally add to) the album depending on permissions granted

### Partner Sharing

Immich supports "partner" sharing where two users' full libraries (or a subset) are visible to each other — useful for couples/family sharing without merging accounts.

## Settings and Configuration

### Account Settings

- **Profile picture**
- **Password change**
- **API Keys**: create/revoke keys for automation (see API documentation)
- **Storage usage**: see how much space your uploads are using

### Admin Settings (Admin Users Only)

- **User management**: create/delete users, set storage quotas
- **Job status**: view background job queues (thumbnail generation, face detection, smart search indexing, library scan) and manually trigger re-runs
- **Library settings**: external library paths, storage template (file naming/folder structure on disk)
- **Server settings**: trash retention period (`30` days in this deployment), OAuth configuration, map tile sources

## Trash and Deletion

- Deleting an asset moves it to **Trash** first; it is permanently removed after the configured retention period (`30` days in this deployment)
- Assets can be restored from Trash before permanent deletion
- Permanent deletion can also be forced immediately if needed

## Advanced Features

### Duplicate Detection

Immich flags potential duplicate uploads (same device asset ID or perceptual match) so you can review and clean up before they clutter the timeline.

### Memories

A "Memories" feature surfaces photos from the same date in previous years, similar to other photo platforms.

### External Libraries

Admins can mount existing photo directories as "external libraries" that Immich indexes without re-uploading/copying the files — useful for large pre-existing collections kept on NAS storage.

## Troubleshooting

### Common Issues

**Issue**: Mobile app backup is stuck or slow
**Solution**: Check Wi-Fi connectivity, confirm the app has background-refresh permission, and check the admin Jobs page for a backed-up upload queue.

**Issue**: Faces not being detected on recently uploaded photos
**Solution**: Face detection runs as an async background job — check **Admin → Jobs → Face Detection** queue status; large batches of uploads can take time to process.

**Issue**: Smart search returns no/irrelevant results
**Solution**: Confirm the Smart Search Indexing job has processed the relevant assets (new uploads need to be indexed before they're searchable by CLIP).

**Issue**: Can't log in / "Server not reachable" in mobile app
**Solution**: Confirm the server URL is exactly `https://immich.conderhome.xyz` and that the ingress/TLS certificate is valid.

### Getting Help

- Check the architecture and deployment guides in this docs set for infrastructure-level issues
- Check `kubectl logs` on `immich-server` / `immich-machine-learning` pods for application-level errors (see deployment guide)
- Upstream Immich documentation and GitHub issues for application bugs

## Best Practices

- Enable background backup on all personal devices so new photos are captured automatically
- Periodically review the **Duplicates** view to keep storage usage down
- Use albums + shared links rather than exporting/emailing originals when sharing with family
- Let the Jobs queue finish processing after large bulk uploads before expecting full search/face results

---

*This documentation is part of the Immich deployment docs for this homelab cluster. For the latest upstream features, consult the official Immich documentation.*
