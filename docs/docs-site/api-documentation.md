---
title: Docs Site API Documentation
date: 2026-10-23
author: Hermes Docs Agent
tags: [api, documentation, mkdocs, static-site]
---

# Docs Site API Documentation

## Overview

Docs Site has no application API. It is a static MkDocs Material site
served by stock nginx — there is no backend process, no database, and no
request handling beyond nginx serving pre-built HTML, CSS, and JS files.
This page exists for consistency with the other apps' documentation sets
and to document the one "interface" this app actually has: its published
container image.

## Interface: The GHCR Image

The only programmatic surface of this app is the built artifact consumed by
the cluster's `HelmRelease`.

**Image**: `ghcr.io/helloworld44-89/homelab-docs`
**Registry**: GitHub Container Registry (GHCR), **public** package
**Tag**: `latest` only — no version-pinned tags are published

### Pulling the image

```bash
docker pull ghcr.io/helloworld44-89/homelab-docs:latest
```

No authentication is required to pull — the GHCR package is public, since
it contains only rendered static documentation already intended for
internal consumption, not secrets.

### Running it standalone

```bash
docker run --rm -p 8080:80 ghcr.io/helloworld44-89/homelab-docs:latest
# then open http://localhost:8080
```

### Image contents

- `/usr/share/nginx/html` — the MkDocs-built static site (HTML/CSS/JS,
  search index)
- Base: `nginx:alpine`
- Listens on port **80**
- No environment variables, no config file, no volumes expected — the
  image is fully self-contained and immutable per build

### What is NOT in the image

- No source markdown beyond what MkDocs rendered into HTML
- No git history, no `.git` directory
- No `INFRA_REPO_PAT` or any other build-time secret (BuildKit secret
  mounts never persist into image layers)
- No Python, pip packages, or MkDocs toolchain (those exist only in the
  discarded build stage)

## HTTP Surface (as served by nginx)

Since there's no app logic, the only "endpoints" are whatever static paths
MkDocs generated, mirroring the `nav:` structure in
`docs-build/mkdocs.yml` — e.g. `/`, `/audiofile/architecture/`,
`/monitoring/monitoring-setup/`, `/proxmox-ansible/`, etc. There is no JSON
API, no `/health` endpoint beyond nginx's default `/` response (used
directly by the Kubernetes liveness/readiness probes — see the deployment
guide), and no webhook support.

## Versioning

There is no semantic versioning for this app. The published image is
always `:latest`, rebuilt on every qualifying push or the daily cron, and
Flux's `HelmRelease` always deploys whatever `:latest` currently resolves
to. To "roll back," revert the offending commit in `docs/**` or
`docs-build/**` and let the pipeline rebuild — there is no mechanism to pin
or redeploy a specific prior image digest through this HelmRelease as
configured.
