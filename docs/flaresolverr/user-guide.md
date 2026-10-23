---
title: FlareSolverr User Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [user-guide, documentation, flaresolverr, prowlarr, troubleshooting]
---

# FlareSolverr User Guide

## Introduction

FlareSolverr has **no user interface and no direct end-user workflow**. It is a headless Cloudflare/CAPTCHA-challenge-solving proxy that exists purely to help Prowlarr talk to indexer sites that would otherwise block Prowlarr's plain HTTP requests. You will never open a browser tab to "use" FlareSolverr. Instead, this guide covers how to confirm it's working, how to wire it into Prowlarr, and how to troubleshoot it when an indexer test fails.

If you came here expecting a dashboard: there isn't one. Everything you need to know about FlareSolverr's state is visible through Prowlarr or `kubectl`.

## What FlareSolverr Actually Does

Some indexer sites sit behind Cloudflare's "Checking your browser..." interstitial. A normal HTTP client (like Prowlarr's built-in fetcher) gets stuck on that page and never sees real search results. FlareSolverr runs a real headless Chromium browser that loads the page, waits for Cloudflare's JavaScript challenge to pass, and hands the resulting cookies/HTML back to Prowlarr so Prowlarr can complete the request itself.

You only need to care about FlareSolverr for the specific indexers in Prowlarr that are flagged as requiring it.

## Checking Whether FlareSolverr Is Configured in Prowlarr

1. Open Prowlarr at `https://prowlarr.conderhome.xyz`
2. Go to **Settings → Indexers → Indexer Proxies**
3. Confirm there is an entry of type **FlareSolverr** pointing at:
   `http://flaresolverr.media.svc.cluster.local:8191`
4. Go to **Settings → Indexers**, open an indexer known to require Cloudflare bypass (commonly flagged in the indexer's description/tags), and verify its **Tags** include the proxy tag associated with that FlareSolverr proxy entry

If the proxy entry is missing, add it: **Settings → Indexers → Indexer Proxies → +**, Type = FlareSolverr, Tag it something like `flaresolverr`, Host = `flaresolverr.media.svc.cluster.local`, Port = `8191`. Then add that tag to any indexer that needs it.

## Verifying FlareSolverr Is Working (No UI — Use Prowlarr's Indexer Test)

This is the primary "is it working" workflow, since there's no FlareSolverr UI to check directly.

1. In Prowlarr, go to **Settings → Indexers**
2. Find an indexer tagged with the FlareSolverr proxy
3. Click the **Test** (flask/beaker icon) button on that indexer row
4. **Success**: the test passes with no error — FlareSolverr successfully solved the challenge and Prowlarr parsed results
5. **Failure**: Prowlarr surfaces an error message in the UI. Common ones and what they mean:

| Prowlarr error text (paraphrased) | Likely cause |
|---|---|
| "Unable to connect to indexer" / connection refused | FlareSolverr pod is down or Service/DNS is broken — check pod status |
| "Error solving the challenge. Timeout..." | Challenge took longer than FlareSolverr's timeout, or the site changed its challenge type |
| CAPTCHA/hCaptcha-related failure | Site is using an interactive CAPTCHA, not just a JS challenge — this deployment has `CAPTCHA_SOLVER: none`, so these are **not supported** and will always fail |
| Test passes but search returns zero results | FlareSolverr itself worked; issue is in Prowlarr's indexer parser, not FlareSolverr |

You can also trigger a manual search for that indexer under **Search** in Prowlarr as a secondary confirmation that real-world queries work end-to-end, not just the synthetic test.

## Confirming the Service Is Up (Cluster-Level Check)

If Prowlarr's indexer test fails with a connection-level error (not a challenge-timeout error), check the pod directly:

```bash
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl get pods -n media -l app.kubernetes.io/name=flaresolverr"
```

Expect a single pod, `Running`, `1/1 Ready`. If it's `CrashLoopBackOff` or `Pending`, see Troubleshooting below.

## Troubleshooting

### Indexer test fails with a timeout, but the pod is Running

**Cause**: the target site's Cloudflare challenge is taking longer than FlareSolverr's `maxTimeout`, or the site escalated to a harder challenge (e.g. a CAPTCHA that needs `CAPTCHA_SOLVER`).

**What to check**:
```bash
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl logs -n media -l app.kubernetes.io/name=flaresolverr --tail=100"
```
Look for the specific URL being fetched and whether Chromium reports a timeout or a crash. If the site now requires an interactive CAPTCHA, FlareSolverr with `CAPTCHA_SOLVER: none` cannot pass it — that indexer is effectively unsupported until a solver is configured (out of scope for this deployment).

### Pod is CrashLoopBackOff or OOMKilled

**Cause**: headless Chromium is memory-hungry, especially with multiple persistent sessions open simultaneously. The current limit is `1Gi` memory / `1` CPU.

**What to check**:
```bash
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl describe pod -n media -l app.kubernetes.io/name=flaresolverr"
```
Look at the `Last State` / `Reason` for the container. If `OOMKilled`, consider raising the memory limit in `apps/media/flaresolverr/helmrelease.yml` and letting Flux reconcile.

### Everything looks healthy but Prowlarr still can't reach it

**Cause**: usually a DNS/Service mismatch rather than FlareSolverr itself.

**What to check**:
```bash
ssh hermes-infra@192.168.5.65 "KUBECONFIG=~/.kube/config kubectl get svc flaresolverr -n media"
```
Confirm the Service exists with port `8191` and that Prowlarr's proxy config uses `flaresolverr.media.svc.cluster.local:8191` (not `localhost` or an external hostname — FlareSolverr has no Ingress and is not reachable from outside the cluster or from a browser).

### After a Flux reconcile / pod restart, a previously-working indexer starts failing intermittently

**Cause**: FlareSolverr's persistent sessions are in-memory only. A pod restart drops them silently; the next call per session simply re-solves from scratch (slower, but shouldn't error). If it's erroring instead, check logs as above — it may be a coincidental timing issue with the challenge, not the restart itself.

## Best Practices

- Only tag indexers that actually require Cloudflare bypass with the FlareSolverr proxy — routing every indexer through it adds unnecessary latency and load to the single FlareSolverr pod
- Don't expect CAPTCHA-protected (not just Cloudflare-JS-challenge) indexers to work — `CAPTCHA_SOLVER` is explicitly `none` here
- If an indexer consistently times out, try raising its FlareSolverr proxy timeout in Prowlarr before assuming FlareSolverr itself is broken
- There's nothing to "update" from an end-user perspective in FlareSolverr — the image tag is pinned to `latest`, so fixes/improvements arrive on the next scheduled image pull rather than through any user action

## Getting Help

- **Logs**: `kubectl logs -n media -l app.kubernetes.io/name=flaresolverr`
- **Prowlarr indexer test**: the fastest signal for "is this indexer's Cloudflare bypass working right now"
- **Upstream project**: https://github.com/FlareSolverr/FlareSolverr (issues/README for challenge-type support questions)

---

*This guide covers the only realistic "usage" of FlareSolverr in this cluster: confirming it's healthy and troubleshooting it through Prowlarr, since it has no direct end-user interface of its own.*
