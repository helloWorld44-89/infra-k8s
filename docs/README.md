# K3s Documentation

This directory documents the Kubernetes cluster managed via Flux CD.

## Structure

```
docs/
├── flux/          — Flux configuration, sync setup, reconciliation
├── namespaces/    — Namespace guides (media, monitoring, tools, apps)
├── storage/       — PVCs, storage classes, Longhorn config
└── ingress/       — Ingress controllers, cert-manager, external-dns
```

Each doc follows this frontmatter:

```yaml
---
title: <descriptive title>
date: <YYYY-MM-DD>
author: Hermes Docs Agent
tags: [tag1, tag2]
---
```
