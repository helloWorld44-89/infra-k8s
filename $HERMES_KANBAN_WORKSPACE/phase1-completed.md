# Phase 1: Prometheus Scrape Target Cleanup - COMPLETED

## What Was Done

### 1. Git Changes Pushed (Already Committed, Just Needed Push)
Commit `b06112a` in `helloWorld44-89/infra-k8s` contained all necessary target removals:
- Removed prod-vault (.35) - destroyed
- Removed prod-hermes-01 (.50) - non-existent host 
- Removed fedsr02 (.27) - unreachable since 2026-10-05
- Added prod-agents-01 (.99) - the actual Hermes host
- Commented out Pi-hole servers (.5/.6) - aspirational, lacking node_exporter

**Action:** Pushed commit to GitHub

### 2. Flux GitOps Reconciliation
- Triggered Flux to sync the git repository
- Updated revision: `main@sha1:b06112a`
- Triggered Prometheus HelmRelease to apply changes
- Restarted Prometheus pod to ensure clean config reload

### 3. Firewall Fix for prod-agents-01
**Issue Found:** UFW only allowed port 9100 from .48 (old monitoring server), not from k3s cluster

**Action:**
```bash
sudo ufw allow from 192.168.5.0/24 to any port 9100 proto tcp comment 'Prometheus node_exporter from LAN'
```

## Final Verification

### DOWN Targets Before Fix: 6
- pisrv01 (.5) - ❌ REMOVED
- pisrv02 (.6) - ❌ REMOVED  
- fedsr02 (.27) - ❌ REMOVED
- prod-vault (.35) - ❌ REMOVED
- prod-hermes-01 (.50) - ❌ REMOVED
- winsrv02 (.26) - Still DOWN (separate issue)

### DOWN Targets After Fix: 1
- winsrv02 (.26) - Windows server timeout (pre-existing issue, unrelated to decommissioned targets)

### All Homelab Targets Status
```
✅ Balerion (192.168.5.10:9100) - Proxmox
✅ Smaug (192.168.5.12:9100) - Proxmox
✅ toothless (192.168.5.11:9100) - Proxmox
✅ prod-ubsrv-01 (192.168.5.40:9100) - Docker
✅ prod-ubsrv-02 (192.168.5.43:9100) - Docker
✅ prod-ubsrv-03 (192.168.5.42:9100) - Docker
✅ dev-ubsrv-01 (192.168.5.45:9100) - Docker
✅ prod-debsrv (192.168.5.48:9100) - Docker
✅ prod-agents-01 (192.168.5.99:9100) - Hermes agent host (NEW)
❌ winsrv02 (192.168.5.26:9182) - Windows (pre-existing timeout issue)
```

## Impact
- Eliminated 5 false-positive DOWN alerts for decommissioned/non-existent hosts
- Added proper monitoring for the actual Hermes agent host (.99)
- Fixed firewall to allow k3s Prometheus to scrape .99

## winsrv02 Investigation Notes
The Windows server (.26) timeout is a separate issue:
- This target was already DOWN before our changes
- Uses windows_exporter on port 9182 (not standard node_exporter on 9100)
- Requires separate investigation (Windows firewall, service status, network connectivity)
- NOT caused by decommissioned target cleanup

## Phase 2 Next Steps
Alert rule queries need fixing to include instance labels in annotations so `[no value]` placeholders are replaced with actual hostnames when data exists.
