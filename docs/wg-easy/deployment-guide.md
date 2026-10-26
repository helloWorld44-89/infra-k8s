# wg-easy Deployment Guide

## Prerequisites

- Kubernetes cluster (k3s v1.35+)
- Traefik ingress with wildcard TLS
- **Host network required** (pinned to single node)
- `NET_ADMIN` + `SYS_MODULE` capabilities
- kubectl / Flux access
- Router port forward: UDP 51820 → master node IP

## Deploy via Flux (GitOps)

```bash
cd /home/jconder44/homelab/infra-k8s
git add apps/tools/wg-easy/
git commit -m "Deploy wg-easy"
git push
```

## Manual Deploy

```bash
kubectl apply -k apps/tools/wg-easy/
```

## Configuration

### HelmRelease (`apps/tools/wg-easy/helmrelease.yml`)

```yaml
image:
  repository: ghcr.io/wg-easy/wg-easy
  tag: latest

# REQUIRED: Host network for WireGuard kernel access
hostNetwork: true
dnsPolicy: ClusterFirstWithHostNet

# Pin to master node (has public IP via router port forward)
nodeSelector:
  kubernetes.io/hostname: prod-k3s-master

# Capabilities for WireGuard
capabilities:
  add:
    - NET_ADMIN
    - SYS_MODULE

env:
  WG_HOST: "wg-easy.conderhome.xyz"  # Public hostname
  PASSWORD: "${WG_EASY_PASSWORD}"    # From secret
  WG_DEFAULT_ADDRESS: "10.0.0.1/24"
  WG_DEFAULT_DNS: "1.1.1.1"
  WG_DEFAULT_MTU: "1420"
  WG_PERSISTENT_KEEPALIVE: "25"
  UI_TRAFFIC_STATS: "true"

# Ports (hostNetwork = host ports)
ports:
  wireguard:
    port: 51820
    protocol: UDP
  webui:
    port: 51821
    protocol: TCP

persistence:
  config:
    type: hostPath  # Or PVC
    path: /etc/wireguard
    # hostPath:
    #   path: /etc/wireguard
    #   type: DirectoryOrCreate

resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 500m
    memory: 256Mi

probes:
  liveness:
    path: /
    port: 51821
    initialDelaySeconds: 10
  readiness:
    path: /
    port: 51821
    initialDelaySeconds: 10

ingress:
  webui:
    host: wg-easy.conderhome.xyz
    port: 51821
    tls: wildcard-conderhome-xyz-tls
```

### Required Secret

```bash
kubectl create secret generic wg-easy-secrets -n tools \
  --from-literal=WG_EASY_PASSWORD="<admin-password>" \
  --dry-run=client -o yaml | kubectl apply -f -
```

### PVC (Optional - can use hostPath)

```yaml
# PVC for config persistence
persistence:
  config:
    enabled: true
    storageClass: local-path
    size: 1Gi
```

## Router Configuration

**Port Forward** (on router/firewall):
- UDP 51820 → 192.168.5.65 (k3s master)
- TCP 51821 → 192.168.5.65 (optional, Web UI via Traefik)

**Firewall**: Allow inbound UDP 51820

## Post-Deploy Verification

```bash
# Check pod
kubectl get pods -n tools -l app.kubernetes.io/instance=wg-easy

# Check logs
kubectl logs -n tools -l app.kubernetes.io/instance=wg-easy --tail=30

# Verify WireGuard interface
kubectl exec -n tools -it <pod> -- wg show

# Web UI
curl -k https://wg-easy.conderhome.xyz/
```

## Migration from Docker (Historical)

Previous: Docker on 192.168.5.43
Critical: **Preserve exact private key** for zero-reconfig migration

1. `docker stop wg-easy`
2. Tar `/etc/wireguard` (6.4KB)
3. Copy to k3s master, extract into PVC via node-debugger
4. Verify: `wg pubkey < privatekey` matches original public key
   - Original: `HEjZuNnIB8dvJECJM+QM4TAbUfGRAv+OuYJhatW56Ao=`
4. Deploy with same key
5. DNS: `wg-easy.conderhome.xyz` → 192.168.5.65
6. Update router port forward to 192.168.5.65
7. Test: Cellular VPN connects with existing peer configs
8. Stop old Docker container

## Backup

```bash
# Config + keys (NEVER expose private keys in backups!)
kubectl exec -n tools <pod> -- tar czf /tmp/wg-backup.tar.gz -C /etc/wireguard .
kubectl cp tools/<pod>:/tmp/wg-backup.tar.gz ./wg-backup-$(date +%F).tar.gz
```

**Critical**: Private keys must never leave server in plaintext backups.

## Upgrade

1. Update image tag in helmrelease.yml
2. `git commit -m "Upgrade wg-easy to <tag>"`
3. `git push`
4. Flux rolls out (pod restart on same node)
5. Verify: `wg show`, peer connections work

## Troubleshooting

| Symptom | Fix |
|---|---|
| Pod not scheduling | Check nodeSelector matches `prod-k3s-master` |
| WireGuard not loading | Check NET_ADMIN/SYS_MODULE capabilities |
| No handshake | Check UDP 51820 port forward, firewall |
| Peers can't connect | Check `wg show`, verify public key matches |
| Pod on wrong node | hostNetwork + nodeSelector required |
| Config lost | Check PVC mount, hostPath permissions |

## Key Migration Critical Path

**Zero client reconfig requires EXACT private key match**:

```bash
# Verify key match
# Original Docker:
docker exec wg-easy wg pubkey < /etc/wireguard/privatekey

# New k3s:
kubectl exec -n tools <pod> -- wg pubkey < /etc/wireguard/privatekey

# Both MUST output: HEjZuNnIB8dvJECJM+QM4TAbUfGRAv+OuYJhatW56Ao=
```

## Monitoring

- **Health**: `/` (liveness/readiness)
- **WireGuard**: `wg show` (peers, handshakes, transfer)
- **Logs**: stdout/stderr
- **Alerts**: Pod crashloop, interface down, peer handshake stale