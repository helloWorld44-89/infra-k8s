---
title: ReadMeABook User Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [user-guide, documentation, audiobook, kubernetes]
---

# ReadMeABook User Guide

## Introduction

ReadMeABook is the household's audiobook request system — think Overseerr, but for audiobooks. Instead of browsing torrent sites or usenet manually, you search for or swipe through audiobook recommendations, submit a request, and ReadMeABook finds it, downloads it, merges/organizes the files, and drops the finished audiobook into the shared library for Plex or Audiobookshelf to pick up.

## Getting Started

### Accessing ReadMeABook

Open a browser to:

```
https://audiobook.conderhome.xyz
```

Traffic is served through Traefik with the cluster's shared wildcard TLS certificate, so the connection is HTTPS by default — no extra setup needed.

### First-Time Setup (admin)

The app ships a guided setup wizard for first run, which:

1. Walks through connecting **Plex** and/or **Audiobookshelf** (server URL + token, library selection)
2. Walks through connecting **Prowlarr** (indexer search aggregator)
3. Walks through connecting download clients — **qBittorrent** (torrents) and/or **SABnzbd** (usenet)
4. Tests each connection live before letting you continue
5. Optionally configures an AI provider (OpenAI, Claude, or a local LLM) for the BookDate recommendation engine

In this homelab, this has already been completed by the admin — most users will just log in and start requesting books.

## Requesting Audiobooks

### Method 1: Search

1. Use the search bar to look up a book by title or author (metadata is Audible-backed)
2. Pick the correct edition/narrator from the results
3. Click **Request**
4. If admin approval is enabled for your account, the request waits for approval; otherwise it moves straight to the acquisition queue

### Method 2: BookDate (swipe recommendations)

1. Open the **BookDate** tab
2. The app surfaces AI-generated audiobook recommendations one at a time
3. Swipe (or click) **right** to request a book, **left** to skip it
4. Your swipe history feeds back into future recommendations

### What Happens After You Request

```
Request submitted → (admin approval, if required) → Prowlarr searches indexers
   → download starts via qBittorrent/SABnzbd → files land in the downloads area
   → multi-file downloads are merged into a single chapterized M4B
   → optional e-book sidecar (EPUB/PDF) is fetched
   → finished audiobook is organized into the shared library
   → Plex/Audiobookshelf picks it up on its next library scan
```

There's no fixed ETA — it depends on indexer availability and download speed. Check the **Requests** page for live status (searching, downloading, processing, available).

## Where the Audiobook Ends Up

ReadMeABook does **not** play audio itself — it's an acquisition tool, not a player. Once a request is marked complete:

- Listen in **Plex** or **Audiobookshelf**, whichever this household uses as the media server
- The file lives on the shared `audiobooks-pvc` library volume, same place every other audiobook lives

## Admin Features

If you have admin access:

- **Approval workflow**: approve or reject pending requests from other users
- **Settings pages**: manage Plex/Audiobookshelf connections, Prowlarr, download clients, AI provider, OIDC login, and notification integrations (Discord, Pushover)
- **Manual Import**: a configurable host folder can be mapped as a "Book Drop" so existing files can be picked up and organized without going through the request/download pipeline
- **Notifications**: configure event subscriptions (e.g. request completed, download failed) to Discord or Pushover, with a test-send button

## Troubleshooting

### Request stuck in "searching"

**Cause**: Prowlarr isn't finding the title on any connected indexer.
**Solution**: Check Prowlarr connectivity in Settings, or try requesting a different edition/narrator — availability varies a lot by release.

### Request stuck in "downloading" for a long time

**Cause**: Slow or stalled torrent/NZB source.
**Solution**: Check qBittorrent/SABnzbd directly for the transfer's real status; some sources are just slow.

### Book never shows up in Plex/Audiobookshelf after "complete"

**Cause**: Usually a library scan hasn't run yet on the media server side.
**Solution**: Trigger a manual library scan in Plex/Audiobookshelf; ReadMeABook writes the file, it doesn't force the media server to re-index.

### Can't log in / app unreachable

**Cause**: Pod restart in progress, or a bad `latest` image pull.
**Solution**: This is a homelab app — ping the admin. Behind the scenes: the deployment uses `imagePullPolicy: Always` with the `latest` tag, so a cluster-side Flux reconcile or pod restart can occasionally pick up a new upstream build; if the app is misbehaving right after a restart, that's the first thing to check.

## Notes for This Deployment

- Single-instance deployment (not scaled out) — one pod, one set of embedded Postgres/Redis, no HA.
- Your request history and app settings live in a dedicated Postgres database that is private to this app (not shared with other services in the cluster).
- Downloaded/organized files pass through NFS-backed shared storage (`downloads-pvc`, `audiobooks-pvc`) that's also used by other media apps in the household stack.

---

*This guide reflects the ReadMeABook deployment running at `audiobook.conderhome.xyz` in this homelab cluster. For deeper upstream feature documentation, see the project repository (kikootwo/ReadMeABook on GitHub).*
