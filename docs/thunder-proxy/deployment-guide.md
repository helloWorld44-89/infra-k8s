# Thunder Proxy Deployment Guide

## Prerequisites

- Kubernetes cluster (k3s v1.35+)
- Traefik ingress with wildcard TLS
- **Traefik config**: `allowExternalNameServices=true` (required!)
- Proxmox host reachable from k3s nodes (192.168.5.10:8006)
- kubectl / Flux access
- Pi-hole DNS for `thunder.conderhome.xyz`

## Deploy via Flux (GitOps)

```bash
cd /home/jconder44/homelab/infra-k8s
git add apps/tools/thunder-proxy/
git commit -m "Add Thunder Proxy for Proxmox UI"
git push
```

## Manual Deploy

```bash
kubectl apply -k apps/tools/thunder-proxy/
```

## Configuration

### Resources (`apps/tools/thunder-proxy/thunder-proxy.yaml`)

```yaml
# ExternalName Service - points Traefik to Proxmox host
apiVersion: v1
kind: Service
metadata:
  name: thunder-proxmox
  namespace: tools
spec:
  type: ExternalName
  externalName: 192.168.5.10
  ports:
    - port: 8006
---
# ServersTransport - trust Proxmox self-signed cert
apiVersion: traefik.io/v1alpha1
kind: ServersTransport
metadata:
  name: thunder-proxmox-transport
  namespace: tools
spec:
  insecureSkipVerify: true
---
# IngressRoute - route thunder.conderhome.xyz to Proxmox
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: thunder-proxmox
  namespace: tools
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`thunder.conderhome.xyz`)
      kind: Rule
      services:
        - name: thunder-proxmox
          port: 8006
          scheme: https
          serversTransport: thunder-proxmox-transport
  tls:
    secretName: wildcard-conderhome-xyz-tls
```

### Kustomization (`apps/tools/thunder-proxy/kustomization.yaml`)

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - thunder-proxy.yaml
```

### Add to Tools Kustomization

```yaml
# apps/tools/kustomization.yaml
resources:
  ...
  - thunder-proxy
  ...
```

## Critical: Traefik Configuration

**Required**: Traefik must allow ExternalName services.

Edit HelmChartConfig `traefik` in `kube-system`:

```yaml
# helmchartconfig traefik -n kube-system
spec:
  valuesContent: |-
    certResolvers:
      letsencrypt:
        email: justin.conder44@icloud.com
        tlsChallenge: true
        storage: /data/acme.json
    additionalArguments:
      - "--providers.kubernetescrd.allowExternalNameServices=true"
      - "--providers.kubernetesingress.allowExternalNameServices=true"
```

Apply:
```bash
kubectl apply -f /tmp/traefik-config.yaml
# Triggers Traefik helm-controller job, restarts pod
```

Verify in Traefik pod args:
```bash
kubectl get deployment -n kube-system traefik -o jsonpath='{.spec.template.spec.containers[0].args}' | grep allowExternalNameServices
```

## DNS Configuration

Add to Pi-hole (both instances):
- `thunder.conderhome.xyz` → 192.168.5.65 (k3s master/Traefik)

Force nebula-sync:
```bash
kubectl delete pod -n tools -l app.kubernetes.io/instance=nebula-sync
```

## Post-Deploy Verification

```bash
# Check resources
kubectl get svc,ingressroute,serverstransport -n tools | grep thunder

# Test HTTP
curl -k https://thunder.conderhome.xyz/
# Should load Proxmox login page (200)

# Test API
curl -k https://thunder.conderhome.xyz/api2/json/version
```

## Troubleshooting

| Symptom | Fix |
|---|---|
| 502 Bad Gateway | Traefik can't reach 192.168.5.10:8006 |
| 404 Not Found | IngressRoute match wrong, check Host() rule |
| Cert error (browser) | wildcard-conderhome-xyz-tls secret missing in tools namespace |
| Backend cert error | ServersTransport `insecureSkipVerify: true` missing |
| 404 on API paths | IngressRoute match needs `PathPrefix(/)` not exact |

## Key Requirements

1. **Traefik `allowExternalNameServices=true`** - Without this, Traefik refuses to route to ExternalName services (security feature)
2. **ServersTransport `insecureSkipVerify: true`** - Proxmox uses self-signed cert on 8006
3. **Wildcard TLS secret in `tools` namespace** - Mirrored via reflector from cert-manager
4. **Network connectivity** - k3s nodes must reach 192.168.5.10:8006
5. **DNS** - Both Pi-holes resolve `thunder.conderhome.xyz` to 192.168.5.65

## Monitoring

- **Traefik metrics**: Check Traefik dashboard/metrics for route hits
- **Proxmox**: Direct monitoring on Proxmox host
- **Alerts**: Traefik route health, Proxmox API reachability