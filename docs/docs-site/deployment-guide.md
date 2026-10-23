---
title: Docs Site Deployment Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [deployment, mkdocs, kubernetes, flux, ci-cd]
---

# Docs Site Deployment Guide

## Overview

This guide covers the full publish pipeline for the Docs Site app — from a
markdown commit in `infra-k8s` to the live page at
`https://docs.conderhome.xyz` — plus exactly how to add a new doc page to
the navigation. There is no separate manual deployment step anywhere in
this flow; it is entirely push-triggered (or cron-triggered) and Flux/GitOps
managed.

## Prerequisites

- Write access to `infra-k8s` on `main` (or via PR)
- For anything touching the `infra` repo's clone step: the
  `INFRA_REPO_PAT` GitHub Actions secret must already exist on `infra-k8s`
  (a fine-grained PAT, read-only, scoped to just the `infra` repository) —
  this is already configured; you only need to touch it if it's rotated or
  expired
- Flux already reconciling this cluster (no separate setup needed to
  deploy a docs change — Flux watches the `tools` namespace's
  `HelmRelease` already)

## How the Pipeline Works End to End

### 1. Trigger

`.github/workflows/docs-site.yml` runs on:
- push to `main` touching `docs/**`, `docs-build/**`, or the workflow file
  itself
- a daily `0 7 * * *` UTC cron — this exists specifically because the
  `infra` repo's docs are cloned at build time and not vendored into
  `infra-k8s`, so a change made only in `infra` has no cross-repo push
  trigger to fire this build; the cron guarantees it's picked up within 24
  hours regardless
- manual `workflow_dispatch`

### 2. Build

The Action checks out `infra-k8s` only, sets up Docker Buildx, logs into
GHCR with the automatic `GITHUB_TOKEN`, then runs
`docker/build-push-action` against `docs-build/Dockerfile`, passing
`INFRA_REPO_PAT` in as a BuildKit secret (`secrets:
infra_repo_pat=${{ secrets.INFRA_REPO_PAT }}`), never a build-arg.

Inside the Dockerfile:
1. **Stage 1** (`python:3.12-slim` + `git` + `mkdocs-material`): copies
   `docs-build/mkdocs.yml` and this repo's `docs/` directory in; then, with
   the mounted secret, clones `https://github.com/helloWorld44-89/infra.git`
   (depth 1) into `/tmp/infra-src`, copies specific files from it into
   `docs/proxmox-ansible/` (`docs/README.md`,
   `docs/network/corosync-redundancy.md`, `runbooks/README.md`,
   `runbooks/CHANGELOG.md`), deletes the clone, then runs `mkdocs build -d
   /site/built`. **Not** `--strict` — `monitoring-setup.md` has pre-existing
   broken links to docs that were planned but never written; this is a
   known, accepted content gap, not something to "fix" by relaxing the
   build further.
2. **Stage 2** (`nginx:alpine`): copies only `/site/built` from stage 1.
   The final image has no git, no Python, no PAT, no raw markdown beyond
   what MkDocs rendered.

### 3. Push

The built image is pushed as `ghcr.io/helloworld44-89/homelab-docs:latest`
— a single, always-overwritten tag. The GHCR package is public (no secrets
inside the image, so there's no reason to keep it private).

### 4. Deploy (Flux)

`apps/tools/docs-site/helmrelease.yml` is a Flux `HelmRelease` (namespace
`tools`) using the `bjw-s` `app-template` chart, pointed at
`ghcr.io/helloworld44-89/homelab-docs:latest`. Flux reconciles on its
normal interval (`1h` on the `HelmRelease` itself; an `ImagePolicy`/
`ImageRepository` automation is not configured here, so picking up a new
`:latest` digest relies on the HelmRelease's own reconcile tick re-pulling
the tag, or a manual `flux reconcile`).

There is no explicit image-digest pinning or canary step — a new build
simply becomes what `:latest` resolves to, and the next reconcile rolls it
out.

### 5. Serve

The pod (`nginx:alpine`, port 80) passes liveness/readiness probes
(`httpGet / :80`), and the `docs-site` `Service` fronts it on port 80. The
Traefik `Ingress` routes `docs.conderhome.xyz` to that Service, TLS
terminated with the shared `wildcard-conderhome-xyz-tls` secret. Pi-hole's
internal DNS is the only thing resolving that hostname — there is no public
DNS record, so the site is unreachable from outside the LAN by construction,
not by firewall rule.

## Adding a New Doc Page to the Nav

This is the step people most often forget. A new `.md` file under `docs/`
builds and is served at its URL regardless, but it will not appear
anywhere in the site's navigation until it's added to
`docs-build/mkdocs.yml`'s `nav:` list.

1. Add your markdown file under `docs/<app-or-topic>/<page>.md` in this
   repo (path is relative to `docs_dir: docs`).
2. Open `docs-build/mkdocs.yml` and add an entry under the appropriate
   section of `nav:`, e.g.:

   ```yaml
   nav:
     - Home: README.md
     - Audiofile:
         - Architecture: audiofile/architecture.md
         - API Documentation: audiofile/api-documentation.md
         - User Guide: audiofile/user-guide.md
         - Deployment Guide: audiofile/deployment-guide.md
     - Docs Site:
         - Architecture: docs-site/architecture.md
         - API Documentation: docs-site/api-documentation.md
         - User Guide: docs-site/user-guide.md
         - Deployment Guide: docs-site/deployment-guide.md
     - Monitoring:
         - Monitoring Setup: monitoring/monitoring-setup.md
     - Proxmox & Ansible (infra repo):
         - Overview: proxmox-ansible/README.md
         - Corosync Redundancy: proxmox-ansible/network/corosync-redundancy.md
         - Runbooks: proxmox-ansible/runbooks-readme.md
         - Runbook Changelog: proxmox-ansible/runbook-changelog.md
   ```

   Create a new top-level section (like `Docs Site:` above) for a new
   app/topic, or add a line under an existing section for a related page.
3. Commit both the new markdown file and the `mkdocs.yml` change
   **together, in the same commit** as whatever infra change the doc
   covers (see Operational Rules below).
4. Push to `main` — the build picks up the nav change automatically; no
   separate step needed.

A quick sanity check before pushing, if you want to catch nav/path typos
locally: `mkdocs build --strict` from a checkout with `docs/` present will
fail loudly on any file referenced in `nav:` that doesn't exist (note: the
actual CI build intentionally does **not** use `--strict`, for the
unrelated `monitoring-setup.md` reason above — this is only a suggestion
for local pre-flight checking, not something to add to the Dockerfile).

## Operational Rules

### Doc-in-the-Same-Commit (critical)

Since this site auto-publishes whatever is committed under `docs/**` or
`docs-build/**` on `main` — no manual publish step exists — **any infra
change elsewhere in this repo must update its corresponding doc page in
the SAME commit/PR as the infra change**, not as a separate later step. An
out-of-date doc page is a review miss here, not a missing pipeline stage.
If a change doesn't fit any existing page, add a new one (and its `nav:`
entry) rather than leave it undocumented.

### Rotating `INFRA_REPO_PAT`

If the fine-grained PAT used to clone the `infra` repo expires or needs
rotation:
1. Generate a new fine-grained PAT scoped to **read-only** access on just
   the `infra` repository (GitHub → Settings → Developer settings →
   Fine-grained tokens).
2. Update the `INFRA_REPO_PAT` secret under `infra-k8s` → Settings →
   Secrets and variables → Actions.
3. No code change needed — the workflow already references the secret by
   name.
4. Trigger a manual `workflow_dispatch` run (or wait for the next push/
   cron) to confirm the clone step still succeeds.

## Monitoring and Maintenance

### Health Checks

```bash
# From inside the LAN
curl -I https://docs.conderhome.xyz

# Check the HelmRelease and pod status on the k3s cluster
kubectl get helmrelease docs-site -n tools
kubectl get pods -n tools -l app.kubernetes.io/name=docs-site
```

### Checking the Last Build

Check the GitHub Actions run history for `docs-site.yml` in `infra-k8s` —
a failed run means the published site is stale (still serving whatever the
last successful build produced), not broken, since Flux never gets a new
image to pull.

### Forcing a Rebuild / Redeploy

```bash
# Force CI to rebuild + push :latest even with no doc changes
gh workflow run docs-site.yml -R helloWorld44-89/infra-k8s

# Force Flux to re-pull :latest and roll the pod now, instead of
# waiting for the next reconcile interval
flux reconcile helmrelease docs-site -n tools
```

## Troubleshooting

### Build fails on the `infra` clone step

```bash
# Check the Action logs for an auth failure on the git clone
# Common cause: INFRA_REPO_PAT expired or revoked
```
Rotate the PAT (see above) and re-run.

### Build fails on `mkdocs build`

Since the build is **not** `--strict`, a failure here means something more
serious than a broken link — e.g. invalid YAML in `mkdocs.yml`, a missing
required plugin, or a genuinely malformed markdown file. Reproduce locally:

```bash
cd infra-k8s
pip install mkdocs-material
cp docs-build/mkdocs.yml ./mkdocs.yml
mkdocs build -d /tmp/built
```

(Omit the `infra` repo clone step for a local repro — it will simply show
as missing files under `proxmox-ansible/`, which is expected and fine for
isolating an `infra-k8s`-side build failure.)

### Site is up but showing old content

See "My doc change isn't showing up" in the user guide — check the Action
run succeeded and that Flux has reconciled since the new image was pushed.

---

*This documentation is part of the infra-k8s Homelab GitOps repository.*
