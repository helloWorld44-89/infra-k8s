---
title: Homepage Deployment Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [deployment, kubernetes, flux, gitops, dashboard]
---

# Homepage Deployment Guide

## Overview

Homepage is deployed in the `tools` namespace of the k3s cluster, managed entirely through Flux GitOps via a HelmRelease using the shared `bjw-s/app-template` chart. There is no manual `kubectl apply` workflow for normal changes — everything goes through the `infra-k8s` Git repository. This guide covers the cluster layout, how to deploy/upgrade the app itself, and — most importantly for day-to-day use — **how to add a new service tile**.

## Prerequisites

### Cluster Requirements
- Running k3s cluster with Flux v2 installed (`helm.toolkit.fluxcd.io/v2` CRDs) and the `bjw-s` HelmRepository already registered in `flux-system`
- Traefik ingress controller and cert-manager configured (used for `homepage.conderhome.xyz` TLS)
- Namespace `tools` created and reconciled

### Access Requirements
- Write access to the `infra-k8s` Git repository
- `kubectl` access to the cluster for verification (e.g. via `ssh hermes-infra@192.168.5.65` with `KUBECONFIG=~/.kube/config`)

## Repository Layout

```
apps/tools/homepage/
├── helmrelease.yml           # HelmRelease: image, service, ingress, persistence mounts
├── configmap.yml             # legacy "homepage-config" ConfigMap — unused, not mounted
├── kustomization.yaml        # includes configmap.yml, helmrelease.yml, configmaps/
└── configmaps/
    ├── kustomization.yaml
    ├── services.yaml          # ← dashboard content, edited most often
    ├── settings.yaml
    ├── widgets.yaml
    ├── bookmarks.yaml
    ├── docker.yaml            # unused placeholder
    ├── kubernetes.yaml        # unused placeholder
    └── proxmox.yaml           # unused placeholder
```

## Initial Deployment

This app is already deployed; redeploying from scratch (e.g. to a new cluster) is just applying the existing manifests via Flux:

1. Ensure the `tools` namespace and `bjw-s` `HelmRepository` exist and are reconciled.
2. Ensure `apps/tools/homepage` is referenced by the parent Kustomization that Flux watches for the `tools` app tier.
3. Commit/push triggers Flux to reconcile:
   - `HelmRelease/homepage` → installs the `app-template` chart with the values in `helmrelease.yml`
   - The chart creates: Deployment `homepage`, Service `app` (port 3000 as `http`), Ingress `app` (host `homepage.conderhome.xyz`, TLS secret `homepage-tls`)
   - The seven ConfigMaps from `configmaps/` are created and mounted via `subPath` into `/app/config/`
4. Verify:
   ```bash
   ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl get pods -n tools -l app.kubernetes.io/name=homepage"
   ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl get ingress -n tools homepage"
   curl -sI https://homepage.conderhome.xyz
   ```

## How to Add a New Service (most common task)

This is the operation you'll do most often. New services get added to the homelab regularly — here's the exact procedure.

### Step 1 — Decide the group

Open `apps/tools/homepage/configmaps/services.yaml` and pick (or create) the top-level group the new service belongs under: `Media`, `Network`, `Services`, `Infrastructure`, or a new group name (any string works — Homepage just renders it as a new section).

### Step 2 — Check for a built-in widget

Look up the service name at https://gethomepage.dev/widgets/ to see if Homepage already has a native widget `type` for it. Most popular self-hosted apps already do (the existing config uses `plex`, `truenas`, `radarr`, `sonarr`, `prowlarr`, `qbittorrent`, `unifi`, `myspeed`, `pihole`, `omada`, `npm`, `immich`, `authentik`, `proxmox`, `dockhand`, `grafana`, `homeassistant`).

### Step 3 — Add the entry

Append a new service block under the chosen group, following the existing style exactly:

```yaml
- Services:                      # existing group, or a new one
    - My New App:
        icon: my-new-app.png     # check https://github.com/gethomepage/homepage/tree/main/public/icons for an exact match, or use a si-<slug>.png
        href: https://mynewapp.conderhome.xyz
        description: Optional one-liner
        ping: https://mynewapp.conderhome.xyz   # optional reachability check
        widget:
          type: <widget-type-if-one-exists>
          url: https://mynewapp.conderhome.xyz
          key: <api-key>         # or username/password, depending on widget type
```

If there's **no built-in widget** but the service has a simple JSON status/health endpoint, use the generic `customapi` adapter instead (copy the pattern used for the `Hermes` tile):

```yaml
- My New App:
    icon: si-mynewapp.png
    href: https://mynewapp.conderhome.xyz
    widget:
      type: customapi
      url: https://mynewapp.conderhome.xyz/api/health
      mappings:
        - field: status
          label: Status
          format: text
        - field: version
          label: Version
          format: text
```

If the service has neither a built-in widget nor a usable API, just omit `widget:` entirely — `icon`, `href`, and optionally `description`/`ping` are enough for a plain link tile.

### Step 4 — Handle credentials

API keys/usernames/passwords for widgets are currently stored directly in `services.yaml` in plaintext (see **Security Considerations** below). For a new entry, follow the existing convention until the secrets-management gap is addressed:
- Get the API key from the new service's own settings page (most *arr-style apps: Settings → General → API Key)
- Add it as `key:` (or `username`/`password:` for basic-auth-style widgets) directly in the YAML

### Step 5 — Commit, push, verify

```bash
cd infra-k8s
git add apps/tools/homepage/configmaps/services.yaml
git commit -m "homepage: add My New App service tile"
git push
```

Flux reconciles on push (or within `interval: 1h` at worst). Confirm the ConfigMap updated and the dashboard shows the new tile:

```bash
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl get configmap homepage-services -n tools -o jsonpath='{.data.services\.yaml}'" | grep -A5 "My New App"
```

Then load `https://homepage.conderhome.xyz` and hard-refresh. No pod restart is normally required — Homepage reads the YAML on each dashboard load. If the tile still doesn't appear after a minute or two:

```bash
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl rollout restart deployment homepage -n tools"
```

### Removing or editing an existing service

Same flow in reverse: edit or delete the block in `services.yaml`, commit, push.

## Configuration Reference

### Deployment-level settings (`helmrelease.yml`)

| Setting | Value | Notes |
|---------|-------|-------|
| Image | `ghcr.io/gethomepage/homepage:latest` | No pinned version — always pulls latest on pod (re)start |
| `HOMEPAGE_ALLOWED_HOSTS` | `homepage.conderhome.xyz` | Must match the Ingress host or the dashboard will reject the request |
| Service port | `3000` (`http`) | ClusterIP, backed by Traefik Ingress |
| Ingress class | `traefik` | TLS via `homepage-tls` secret |

### ConfigMap mounts

| Container path | ConfigMap | subPath |
|-----------------|-----------|---------|
| `/app/config/services.yaml` | `homepage-services` | `services.yaml` |
| `/app/config/settings.yaml` | `homepage-settings` | `settings.yaml` |
| `/app/config/widgets.yaml` | `homepage-widgets` | `widgets.yaml` |
| `/app/config/bookmarks.yaml` | `homepage-bookmarks` | `bookmarks.yaml` |
| `/app/config/docker.yaml` | `homepage-docker` | `docker.yaml` |
| `/app/config/kubernetes.yaml` | `homepage-kubernetes` | `kubernetes.yaml` |
| `/app/config/proxmox.yaml` | `homepage-proxmox` | `proxmox.yaml` |
| `/app/config/logs` | — (`emptyDir`) | — |

## Monitoring and Maintenance

### Health Checks

```bash
# Pod status
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl get pods -n tools -l app.kubernetes.io/name=homepage"

# Dashboard reachability
curl -sI https://homepage.conderhome.xyz

# Logs
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl logs -n tools -l app.kubernetes.io/name=homepage --tail=100"
```

No `livenessProbe`/`readinessProbe` is currently configured on the Deployment — a hung Next.js process inside a "Running" pod would not be auto-detected by Kubernetes. If you want self-healing on hang, add an HTTP probe against `/` in `helmrelease.yml`'s controller spec.

### Forcing a Config Reload

Homepage generally picks up YAML changes live on the next dashboard request. If a change doesn't seem to take effect:

```bash
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl rollout restart deployment homepage -n tools"
```

Note the Deployment uses `strategy: Recreate`, so this causes a brief (seconds) dashboard outage — the old pod fully terminates before the new one starts.

### Upgrading the Image

The image tag is `latest` with `imagePullPolicy: Always`, so a simple rollout restart pulls whatever upstream has published:

```bash
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl rollout restart deployment homepage -n tools"
```

To pin a specific version instead (recommended if an upstream release ever breaks something), set an explicit tag in `helmrelease.yml`:

```yaml
image:
  repository: ghcr.io/gethomepage/homepage
  tag: v1.x.y
```

## Security Considerations

- **Plaintext secrets in `services.yaml`**: every widget credential (Plex key, TrueNAS/Unifi/Omada/NPM/Dockhand/Grafana passwords, Pi-hole keys, Immich key, Proxmox API token, Home Assistant long-lived token) is stored in plain YAML committed to Git. This is the single biggest gap in the current setup. Mitigations to consider:
  - Move secrets into Kubernetes `Secret` objects and have Flux substitute them via `postBuild.substituteFrom` or SOPS-encrypted values, rather than committing them in the ConfigMap directly.
  - At minimum, ensure the `infra-k8s` repository itself has restricted access, since cloning it currently exposes every integrated service's credentials.
- **No dashboard authentication**: Homepage has no login wall in front of it; it's reachable to anyone on the internal network/VPN who can resolve `homepage.conderhome.xyz`. If broader network exposure is ever considered, put an auth proxy (e.g. Authentik forward-auth) in front of the Ingress first — this would also help since several widgets (Grafana, NPM, Dockhand) currently double as a way to view embedded status for apps that otherwise require their own login.
- **`HOMEPAGE_ALLOWED_HOSTS`**: keep this in sync with the actual Ingress host; a mismatch causes the app to reject requests outright.
- **Legacy `homepage-config` ConfigMap**: `apps/tools/homepage/configmap.yml` defines a `homepage-config` ConfigMap that is not referenced by the current HelmRelease's `persistence` block and has no effect — it's safe to remove in a cleanup pass, but is harmless as-is.

## Troubleshooting

### Dashboard shows a blank/error page after a config change

```bash
# Check for YAML errors in the logs
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl logs -n tools -l app.kubernetes.io/name=homepage --tail=200"
```
Fix the YAML in the repo, commit, push, and let Flux reconcile again (or `kubectl rollout restart` after confirming the ConfigMap updated).

### Flux isn't picking up a config change

```bash
# Check Flux reconciliation state
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl get kustomization -A | grep -i tools"
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl get helmrelease homepage -n tools"
```
Force an immediate reconcile:
```bash
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl annotate kustomization <name> -n flux-system reconcile.fluxcd.io/requestedAt=$(date +%s) --overwrite"
```

### New service tile added but widget shows no data

- Confirm the service's `url` is reachable **from inside the cluster** (not just from your laptop) — test from a debug pod if needed.
- Re-check the API key/credentials against the target service's current settings (keys rotate, especially after a service redeploy).
- Confirm the widget `type` and required fields match the exact schema at https://gethomepage.dev/widgets/ for that service — a typo'd field is silently ignored, not erred.

### Pod stuck in `Recreate` termination

Since `strategy: Recreate` fully tears down the old pod first, a slow-terminating pod (unlikely here, no PVCs) blocks the new one from starting. Check:
```bash
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl get pods -n tools -l app.kubernetes.io/name=homepage -o wide"
```

---

*This deployment guide covers the Conder Home homelab's Homepage instance, managed via Flux GitOps in the infra-k8s repository. For upstream configuration options beyond what's used here, see https://gethomepage.dev/.*
