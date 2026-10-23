---
title: Docs Site Architecture Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [architecture, documentation, mkdocs, meta]
---

# Docs Site Architecture Guide

## Overview

Docs Site is the self-hosted MkDocs Material documentation site for this
homelab's Flux-managed k3s cluster. It is a meta-documentation system: it
renders the markdown living in this repo (`infra-k8s/docs/`) and, at build
time, also pulls in markdown from a second, separate repo (`infra`) so that
Ansible/Terraform/Proxmox documentation can be published alongside the k3s
docs without being duplicated or vendored into either repo.

There is no application logic, database, or API — the entire system is a
static site generator (MkDocs) wrapped in a two-stage Docker build, served
by stock nginx behind the cluster's existing Traefik ingress.

## System Architecture

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                  GitHub Actions (CI)                         │
│                                                                │
│  ┌──────────────┐   ┌───────────────────────┐                │
│  │ Checkout      │   │ docker/build-push-     │                │
│  │ infra-k8s     │──▶│ action (buildx)        │                │
│  └──────────────┘   └───────────┬────────────┘                │
│                                  │ BuildKit secret:            │
│                                  │ INFRA_REPO_PAT               │
└──────────────────────────────────┼──────────────────────────┘
                                   ▼
                     ┌──────────────────────────┐
                     │   docs-build/Dockerfile   │
                     │   (multi-stage build)     │
                     │                            │
                     │ Stage 1: build (python)    │
                     │  - COPY docs/ (this repo)  │
                     │  - git clone infra repo    │
                     │    (docs/, runbooks/)      │
                     │  - mkdocs build            │
                     │                            │
                     │ Stage 2: nginx:alpine       │
                     │  - COPY built site only     │
                     └─────────────┬──────────────┘
                                   │ push
                                   ▼
                   ghcr.io/helloworld44-89/homelab-docs
                          (public GHCR package)
                                   │
                                   ▼ pulled by Flux/k3s
┌──────────────────────────────────────────────────────────────┐
│                     k3s cluster, namespace: tools              │
│                                                                │
│  ┌─────────────┐   ┌─────────────┐   ┌───────────────────┐   │
│  │ HelmRelease  │──▶│ Deployment   │──▶│ nginx:alpine pod   │   │
│  │ docs-site    │   │ (app-template)│  │ serving static HTML│  │
│  └─────────────┘   └─────────────┘   └─────────┬──────────┘   │
│                                                 │              │
│                                    ┌────────────▼───────────┐ │
│                                    │ Service docs-site :80  │ │
│                                    └────────────┬───────────┘ │
│                                                 │              │
│                                    ┌────────────▼───────────┐ │
│                                    │ Traefik Ingress         │ │
│                                    │ host: docs.conderhome.xyz│ │
│                                    │ TLS: wildcard-conderhome- │ │
│                                    │      xyz-tls (shared)    │ │
│                                    └────────────┬───────────┘ │
└──────────────────────────────────────────────────┼─────────────┘
                                                    ▼
                                   Pi-hole internal DNS only
                                   (docs.conderhome.xyz, LAN-only,
                                    no public exposure)
```

### Core Components

#### 1. Build Pipeline (CI)

**GitHub Action** (`.github/workflows/docs-site.yml`)
- Triggers: push to `main` touching `docs/**`, `docs-build/**`, or the
  workflow file itself; a daily `0 7 * * *` UTC cron; manual
  `workflow_dispatch`
- Checks out only `infra-k8s` (this repo) — the `infra` repo is cloned
  separately, inside the Docker build, not by the Action itself
- Logs into GHCR using the automatic `GITHUB_TOKEN` (no separate registry
  credential needed)
- Builds and pushes with `docker/build-push-action`, passing the
  `INFRA_REPO_PAT` GitHub secret through as a BuildKit secret (`secrets:
  infra_repo_pat=...`) rather than a build-arg, so the token never lands in
  an image layer or build history
- Publishes a single tag: `ghcr.io/helloworld44-89/homelab-docs:latest`

**Docker Build** (`docs-build/Dockerfile`, two stages)
- **Stage 1 (`python:3.12-slim`)**: installs `git` + `mkdocs-material`,
  copies `docs-build/mkdocs.yml` and this repo's `docs/` directory, then
  clones the separate `infra` repo at build time (via the mounted
  `infra_repo_pat` secret, never baked into the image) to pull in
  `docs/README.md`, `docs/network/corosync-redundancy.md`,
  `runbooks/README.md`, and `runbooks/CHANGELOG.md` into
  `docs/proxmox-ansible/`. Runs `mkdocs build -d /site/built`
  (intentionally **not** `--strict`, see Known Limitations below).
- **Stage 2 (`nginx:alpine`)**: copies only the built static HTML output
  from stage 1 — the final image carries no Python, no git, no PAT, and no
  source markdown beyond what MkDocs rendered.

#### 2. MkDocs Site Generator

**Configuration** (`docs-build/mkdocs.yml`)
- Theme: `mkdocs-material`, slate/indigo palette
- Features: `navigation.sections`, `navigation.top`, `search.suggest`,
  `content.code.copy`
- Plugin: `search` (built-in full-text search, no external service)
- `nav:` defines the entire site structure explicitly — any markdown file
  not listed here is built but not linked from the nav

#### 3. Runtime Serving

**nginx:alpine**
- Serves the pre-built static HTML/CSS/JS from stage 1; no server-side
  logic, no templating at request time
- Listens on port 80 inside the container

**Kubernetes Deployment** (`apps/tools/docs-site/helmrelease.yml`)
- Flux `HelmRelease` using the shared `bjw-s` `app-template` chart
  (version `3.x`), namespace `tools`
- Single container `app`, image `ghcr.io/helloworld44-89/homelab-docs:latest`
- Liveness/readiness probes: `httpGet / :80`
- Resources: requests `10m`/`32Mi`, limits `100m`/`128Mi` — a static file
  server needs almost nothing
- `Service` `docs-site` on port 80 (`http`)
- Traefik `Ingress`, host `docs.conderhome.xyz`, TLS via the cluster-wide
  `wildcard-conderhome-xyz-tls` secret (same cert reflected into every
  namespace, not a dedicated cert for this app)

### Data Flow Architecture

#### Publish Pipeline

```
1. Commit to docs/** or docs-build/** on main
        ↓
2. GitHub Action triggers → Docker buildx build
        ↓
3. Stage 1: copy infra-k8s docs + clone infra repo docs (PAT-gated)
        ↓
4. mkdocs build → static HTML
        ↓
5. Stage 2: nginx:alpine + static HTML → push to GHCR :latest
        ↓
6. Flux reconciles HelmRelease (interval: 1h, or on image/chart change)
        ↓
7. New pod pulls :latest, passes probes, old pod terminated
        ↓
8. docs.conderhome.xyz serves updated content (LAN-only)
```

There is no separate "publish" step distinct from the build — pushing the
`:latest` tag to GHCR and Flux's reconciliation loop are the entire release
process. Whatever is committed to `main` under the watched paths becomes
live automatically, on the next Action run and the next Flux reconcile.

### Cross-Repo Documentation Model

This is the one architecturally unusual piece of the system: two source
repos, one published site, zero duplication.

- `infra-k8s` (this repo) owns `docs/` directly — these pages are checked
  out normally as part of the Action's own repo checkout.
- `infra` (a separate repo covering Proxmox/Ansible/Terraform) owns its own
  `docs/` and `runbooks/` — these are **cloned fresh inside the Docker
  build** on every build, copied into `docs/proxmox-ansible/` in the build
  context, and never committed or vendored into `infra-k8s`. This means the
  `infra` repo's docs can never drift out of sync with their source by
  construction — there is nothing in `infra-k8s` to go stale.
- The tradeoff: because there's no cross-repo push trigger, a change
  committed only to `infra`'s docs doesn't retrigger this pipeline. The
  daily `0 7 * * *` cron exists specifically to catch that case within 24
  hours even with zero `infra-k8s` activity.
- This requires a scoped credential: `INFRA_REPO_PAT`, a fine-grained GitHub
  PAT with **read-only** access to just the `infra` repository, stored as a
  GitHub Actions secret on `infra-k8s` and passed into the build only as a
  BuildKit secret mount (never a build-arg, never written to a layer).

### Security Architecture

- **No authentication, no public exposure**: the site is reachable only via
  Pi-hole's internal DNS resolving `docs.conderhome.xyz` to a LAN IP; there
  is no port-forward, no public ingress annotation, and no WAN route.
  Anyone off the LAN gets nothing — there's no public DNS record at all.
- **TLS**: terminated at the Traefik ingress using the shared
  `wildcard-conderhome-xyz-tls` certificate, the same wildcard cert used by
  every other `*.conderhome.xyz` service — no dedicated cert issuance for
  this app.
- **Build-time secret handling**: `INFRA_REPO_PAT` is a fine-grained,
  read-only-on-one-repo PAT, passed via Docker BuildKit's `--mount
  type=secret`, which is never persisted into any image layer or the build
  cache's committed history — it only exists transiently in the build
  container's `/run/secrets/`.
- **GHCR image visibility**: the built image is pushed to a **public** GHCR
  package (`ghcr.io/helloworld44-89/homelab-docs`). This is intentional —
  the image itself contains only rendered static HTML of homelab
  documentation already scoped for LAN/internal consumption, not secrets.

### Known Limitations

- `mkdocs build` runs **without** `--strict`. `monitoring-setup.md`
  contains pre-existing broken links to docs that were planned but never
  written (`flux/README.md`, `ingress/README.md`, `monitoring.md`) and to a
  non-markdown config file. These are real content gaps in the source docs,
  not a build problem, and intentionally don't block the pipeline — but
  they're still worth fixing in those source files eventually.
- Any file under `docs/` that isn't listed in `docs-build/mkdocs.yml`'s
  `nav:` is rendered into the site output but has no link pointing to it
  from the navigation — effectively orphaned. Always add new pages to
  `nav:` (see the deployment guide for the exact procedure).

## Technology Stack

- **MkDocs** + **mkdocs-material**: static site generator and theme
- **Python 3.12-slim**: build-stage base image
- **git**: build-stage dependency, used to clone the `infra` repo
- **nginx:alpine**: runtime-stage base image, serves the built HTML
- **Docker multi-stage build** + **BuildKit secrets**: build pipeline
- **GitHub Actions**: CI, scheduled + path-triggered + manual
- **GHCR**: image registry (public package)
- **Flux (HelmRelease)** + **bjw-s app-template chart**: Kubernetes
  deployment
- **Traefik**: ingress controller and TLS termination
- **Pi-hole**: internal-only DNS resolution

## Development Guidelines

### Operational Rule: Doc-in-the-Same-Commit

Because this site auto-publishes whatever is committed under `docs/**` or
`docs-build/**` — with no separate manual publish step — any infra change
elsewhere in this repo that materially affects a documented behavior
(Corosync setup, monitoring stack, an app's deployment, a runbook step,
etc.) must update the corresponding doc page **in the same commit/PR** as
the infra change, not as a later follow-up. An out-of-date doc page here is
a review miss, not a missing pipeline step. If a change doesn't fit any
existing doc page, add a new one under `docs/` rather than leave it
undocumented (and remember to add it to `nav:` — see the deployment guide).

### Adding Documentation

- New pages live under `docs/<app-or-topic>/*.md` in this repo, following
  the same four-document pattern used here (architecture, API/interface,
  user guide, deployment guide) when documenting an app; freeform pages
  (runbooks, monitoring notes) don't need to follow that split.
- Every new page must be added to `docs-build/mkdocs.yml`'s `nav:` or it
  won't be reachable from the site navigation.
