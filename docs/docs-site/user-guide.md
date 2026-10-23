---
title: Docs Site User Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [user-guide, documentation, mkdocs, meta]
---

# Docs Site User Guide

## Introduction

Docs Site is the homelab's internal documentation portal, published at
`https://docs.conderhome.xyz`. This guide is for anyone browsing or
contributing to the docs — not for operating the app itself, since there's
nothing to operate beyond reading and writing markdown.

## Getting Started

### Accessing the Site

1. Be on the home LAN (or VPN'd into it) — the site is **not** reachable
   from outside the network, there is no public DNS record or ingress
   route for it.
2. Open `https://docs.conderhome.xyz` in a browser. DNS resolution works
   automatically via Pi-hole's internal DNS; no `/etc/hosts` entry or
   manual IP is needed.
3. The cert is valid (shared `wildcard-conderhome-xyz-tls` cert) — no
   browser warnings expected.

### What You'll See

- **Home**: the repo's top-level `README.md`
- **Audiofile**: architecture, API, user guide, and deployment guide for
  the AudioFile app
- **Monitoring**: monitoring stack setup notes
- **Proxmox & Ansible (infra repo)**: docs and runbooks pulled in from the
  separate `infra` repo at build time — README, Corosync redundancy notes,
  runbooks README, and runbook changelog
- **Docs Site**: this app's own meta-documentation (architecture, API,
  user guide, deployment guide)

Use the built-in search (top of the page) to full-text search across every
page on the site — it's a static client-side index, no network round trip
needed.

## Reading the Docs

- Navigation sections on the left group pages by app/topic, matching the
  structure in `docs-build/mkdocs.yml`'s `nav:`.
- Code blocks have a copy-to-clipboard button (`content.code.copy`
  feature).
- The theme is dark (slate/indigo) by default — there is no light-mode
  toggle configured.

## Contributing a Doc Change

You don't need to touch this app's code or container to update the site —
only the markdown source.

1. Edit or add a `.md` file under `infra-k8s/docs/<topic>/`.
2. If it's a new page, add an entry for it in `docs-build/mkdocs.yml`'s
   `nav:` (see the deployment guide for the exact syntax) — otherwise it
   renders but has no link pointing to it.
3. **If your change documents an infra change you're making elsewhere in
   this repo, commit the doc update in the same commit/PR as that infra
   change.** This is the house rule for this repo: the site auto-publishes
   whatever lands on `main` under `docs/**` or `docs-build/**`, with no
   separate manual publish step, so a docs update made "later" is just a
   docs update that's wrong until later.
4. Push to `main`. The GitHub Action rebuilds and publishes automatically
   — there is nothing further to click, run, or deploy.
5. Within a few minutes (build + push + Flux's reconcile), the change is
   live at `docs.conderhome.xyz`.

### Updating the `infra` Repo's Docs

Pages under "Proxmox & Ansible (infra repo)" live in a separate repository
(`infra`), not `infra-k8s`. Edit them there, in that repo, normally. They
won't appear on `docs.conderhome.xyz` until the next build here runs — this
happens automatically within 24 hours via the daily `07:00 UTC` cron even
if nothing changes in `infra-k8s` itself, or immediately if any
`infra-k8s` change to `docs/**`/`docs-build/**` happens to trigger a build
in the meantime.

## Troubleshooting

### Site won't load

**Issue**: `docs.conderhome.xyz` doesn't resolve or times out
**Solution**: Confirm you're on the LAN — this is expected behavior from
off-network, not a bug. If on the LAN, check Pi-hole's local DNS entry for
the hostname and that it points at a reachable ingress IP.

### My doc change isn't showing up

**Issue**: Pushed a markdown change but the live site still shows the old
content
**Solution**: Check the GitHub Action run for `docs-site.yml` succeeded
(push must touch `docs/**`, `docs-build/**`, or the workflow file itself to
auto-trigger). Then check Flux has reconciled the `docs-site` HelmRelease
in the `tools` namespace — it polls on a 1h interval by default, or trigger
a manual reconcile. If your change was to `infra`-repo docs only, remember
it needs either an `infra-k8s` trigger or the daily cron to actually
rebuild this site.

### New page doesn't appear in navigation

**Issue**: Added a markdown file but it's not linked anywhere on the site
**Solution**: You forgot to add it to `docs-build/mkdocs.yml`'s `nav:` —
the file still builds and is reachable by direct URL, just not linked.

## Getting Help

- This guide and the architecture/deployment docs for this app, all under
  `docs/docs-site/`
- The `homelab` skill (operational notes, including the doc-in-same-commit
  rule and the cross-repo build model)
- Repo history (`git log -- docs/ docs-build/ .github/workflows/docs-site.yml`)
  for how a given doc page or the pipeline itself evolved

---

*This documentation is part of the infra-k8s Homelab GitOps repository.*
