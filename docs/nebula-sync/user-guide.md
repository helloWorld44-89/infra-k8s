# Nebula Sync User Guide

## Overview

Nebula Sync keeps Pi-hole Local DNS Records (Custom.list) in sync between primary and secondary Pi-hole instances. Runs hourly via cron.

## What It Syncs

- **Source**: Primary Pi-hole (192.168.5.5) - `custom.list`
- **Destination**: Secondary Pi-hole (192.168.5.6) - `custom.list`
- **Scope**: All records in primary's Custom.list
- **Frequency**: Hourly (minute 0)

## Access

No UI. Monitor via:
- **Logs**: `kubectl logs -n tools -l app.kubernetes.io/instance=nebula-sync`
- **CronJob**: `kubectl get cronjob nebula-sync -n tools`

## Manual Sync

Force immediate sync:

```bash
kubectl create job --from=cronjob/nebula-sync manual-sync-$(date +%s) -n tools
kubectl logs -n tools job/manual-sync-<timestamp>
```

## Verifying Sync

1. Check primary: `cat /etc/pihole/hosts/custom.list` (on 192.168.5.5)
2. Check secondary: `cat /etc/pihole/hosts/custom.list` (on 192.168.5.6)
3. Should be identical (after sync runs)

## Adding Records

1. Add record to primary Pi-hole (UI or `/etc/pihole/hosts/custom.list`)
2. Wait for next hourly sync, or trigger manual sync
3. Verify on secondary

## Troubleshooting

| Issue | Fix |
|---|---|
| Sync not running | Check CronJob status: `kubectl get cronjob nebula-sync -n tools` |
| Secondary out of sync | Check logs: `kubectl logs -n tools -l app.kubernetes.io/instance=nebula-sync` |
| API auth failure | Verify API keys in `nebula-sync-secrets` secret |
| Network timeout | Check connectivity between k3s and Pi-hole IPs |
| FTL not reloading | Secondary Pi-hole may need manual restart |

## Logs

```bash
# Recent runs
kubectl logs -n tools -l app.kubernetes.io/instance=nebula-sync --tail=50

# Follow live
kubectl logs -n tools -l app.kubernetes.io/instance=nebula-sync -f
```

## Forcing FTL Reload

If sync ran but secondary didn't reload:

```bash
ssh root@192.168.5.6 "pihole restartdns reload"
```