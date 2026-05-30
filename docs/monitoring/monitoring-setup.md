---
title: K3s Monitoring Setup and Migration Guide
date: 2026-05-29
author: Hermes Docs Agent
tags: [monitoring, k3s, grafana, prometheus, homelab]
---

# K3s Monitoring Setup and Migration Guide

This document provides comprehensive documentation for the Grafana and Prometheus monitoring deployment on the K3s cluster, including installation commands, configuration files, access URLs, backup procedures, and troubleshooting tips.

## Overview

The homelab monitoring stack consists of three main components deployed via Flux CD on the K3s cluster:

- **Prometheus**: Time series database and monitoring system
- **Grafana**: Visualization and dashboarding platform  
- **Uptime Kuma**: Simple uptime monitoring

## Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Prometheus    │    │    Grafana     │    │  Uptime Kuma   │
│    (Port 9090)   │◄───│ (Port 3000)     │    │  (Port 3001)   │
│                 │    │                 │    │                 │
│  Data Storage   │    │ Dashboards/    │    │   Health Checks │
│   (5Gi PV)      │    │  Datasources   │    │  (1Gi PV)      │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └──────────┬─────────────┴───────────┬─────────────┘
                    │                         │
         ┌─────────────────────────────────┐ │
         │      K3s Cluster (monitoring ns) │ │
         └─────────────────────────────────┘ │
                    │                         │
         ┌─────────────────────────────────┐ │
         │       Ingress (Traefik)          │ │
         │                                │ │
         │  grafana.conderhome.xyz         │ │
         │  prometheus.conderhome.xyz      │ │
         │  kuma.conderhome.xyz            │ │
         └─────────────────────────────────┘
```

## Installation and Deployment

### Prerequisites

- K3s cluster running with Traefik ingress controller
- Flux CD installed and configured
- Monitoring namespace created
- Helm repositories configured:
  - `grafana` in `flux-system` namespace
  - `prometheus-community` in `flux-system` namespace
  - `bjw-s` in `flux-system` namespace (for uptime-kuma)

### Deployment Commands

#### 1. Apply Monitoring Stack

```bash
# Navigate to monitoring app directory
cd homelab/infra-k8s/apps/monitoring

# Apply the entire monitoring stack
kubectl apply -k .

# Verify deployment
kubectl get pods -n monitoring
kubectl get helmreleases -n monitoring
```

#### 2. Check Deployment Status

```bash
# Monitor Flux sync status
kubectl get helmreleases -n monitoring -w

# Check pod status
kubectl get pods -n monitoring -w

# Check ingress status
kubectl get ingress -n monitoring
```

## Configuration Files

### Prometheus Configuration

**File**: `homelab/infra-k8s/apps/monitoring/prometheus/helmrelease.yml`

**Key Settings**:
- Chart: `prometheus` version `25.x`
- Persistent Volume: 5Gi
- Ingress: `prometheus.conderhome.xyz`
- AlertManager: Disabled
- PushGateway: Disabled
- Kube-State-Metrics: Enabled

**Scrape Jobs**:
- `homelab-proxmox`: Proxmox nodes (Balerion, Smaug, toothless)
- `homelab-servers`: Ubuntu servers (prod-ubsrv-01/02/03, dev-ubsrv-01, prod-debsrv, prod-vault, prod-hermes-01, fedsr02)
- `homelab-pi`: Raspberry Pi servers (pisrv01, pisrv02)
- `homelab-windows`: Windows server (winsrv02)

**Access URL**: https://prometheus.conderhome.xyz

### Grafana Configuration

**File**: `homelab/infra-k8s/apps/monitoring/grafana/helmrelease.yml`

**Key Settings**:
- Chart: `grafana` version `8.x`
- Persistent Volume: 2Gi
- Ingress: `grafana.conderhome.xyz`
- Admin credentials stored in secret: `grafana-secrets`
- Prometheus datasource pre-configured
- Dashboard sidecar enabled

**Access URL**: https://grafana.conderhome.xyz

### Uptime Kuma Configuration

**File**: `homelab/infra-k8s/apps/monitoring/uptimekuma/helmrealease.yml`

**Key Settings**:
- Chart: `app-template` version `3.x`
- Image: `louislam/uptime-kuma:2`
- Persistent Volume: 1Gi
- Ingress: `kuma.conderhome.xyz`
- Service Port: 3001

**Access URL**: https://kuma.conderhome.xyz

## Access and Login

### Grafana

**URL**: https://grafana.conderhome.xyz

**Login**:
- Username: `GF_SECURITY_ADMIN_USER` (from grafana-secrets)
- Password: `GF_SECURITY_ADMIN_PASSWORD` (from grafana-secrets)

**Initial Setup**:
1. Login with admin credentials
2. Configure data sources (already pre-configured for Prometheus)
3. Import or create dashboards
4. Set up alert notifications

### Prometheus

**URL**: https://prometheus.conderhome.xyz

**Features**:
- Expression browser for querying metrics
- Graph visualization
- Alert management (disabled in current config)
- Status monitoring

### Uptime Kuma

**URL**: https://kuma.conderhome.xyz

**Features**:
- Simple uptime monitoring
- Multiple notification methods
- Status pages
- Heartbeat monitoring

## Backup Procedures

### Grafana Backups

#### 1. Configuration Backup

```bash
# Export Grafana configuration
kubectl exec -n monitoring deployment/grafana -- cp -r /var/lib/grafana/conf /tmp/grafana-conf
kubectl cp -n monitoring deployment/grafana:/tmp/grafana-conf ./grafana-config-backup

# Export dashboards and data sources
kubectl exec -n monitoring deployment/grafana -- grafana-cli --home /var/lib/grafana plugins ls
kubectl exec -n monitoring deployment/grafana -- grafana-cli --home /var/lib/grafana dashboards export all > grafana-dashboards-backup.json
```

#### 2. Persistent Volume Backup

```bash
# Backup PV data
velero backup create grafana-backup --include-namespaces monitoring --selector app.kubernetes.io/name=grafana
```

### Prometheus Backups

#### 1. Configuration Backup

```bash
# Export Prometheus configuration
kubectl exec -n monitoring deployment/prometheus-prometheus-server -- cat /etc/prometheus/prometheus.yml > prometheus-config-backup.yml
```

#### 2. Data Backup

```bash
# Create snapshot
kubectl exec -n monitoring deployment/prometheus-prometheus-server -- /bin/sh -c 'curl -X POST http://localhost:9090/api/v1/admin/tsdb/snapshot'

# Copy snapshot data
kubectl cp -n monitoring deployment/prometheus-prometheus-server:/prometheus/snapshots ./prometheus-snapshots
```

#### 3. Persistent Volume Backup

```bash
# Backup PV data
velero backup create prometheus-backup --include-namespaces monitoring --selector app.kubernetes.io/name=prometheus
```

### Uptime Kuma Backups

```bash
# Export configuration and data
kubectl exec -n monitoring deployment/uptimekuma -- tar -czf - /app/data > uptimekuma-backup.tar.gz

# Persistent volume backup
velero backup create uptimekuma-backup --include-namespaces monitoring --selector app.kubernetes.io/name=uptimekuma
```

## Troubleshooting

### Common Issues

#### 1. Pods Not Starting

```bash
# Check pod status and events
kubectl get pods -n monitoring
kubectl describe pod <pod-name> -n monitoring

# Check PVC status
kubectl get pvc -n monitoring
kubectl describe pvc <pvc-name> -n monitoring
```

#### 2. Ingress Not Working

```bash
# Check ingress status
kubectl get ingress -n monitoring
kubectl describe ingress <ingress-name> -n monitoring

# Check Traefik logs
kubectl logs -n traefik deployment/traefik
```

#### 3. Data Source Connection Issues

**Grafana → Prometheus**:
```bash
# Test Prometheus connectivity
kubectl exec -n monitoring deployment/grafana -- wget -qO- http://prometheus-server.monitoring.svc.cluster.local:9090/api/v1/query?query=up

# Check Prometheus service
kubectl get svc -n monitoring
kubectl describe svc prometheus-server -n monitoring
```

#### 4. Storage Issues

```bash
# Check PV capacity
kubectl get pv -n monitoring

# Check PVC usage
kubectl exec -n monitoring deployment/grafana -- df -h /var/lib/grafana
kubectl exec -n monitoring deployment/prometheus-prometheus-server -- df -h /data
```

### Debug Commands

#### 1. Access Pods for Debugging

```bash
# Access Grafana pod
kubectl exec -it -n monitoring deployment/grafana -- /bin/sh

# Access Prometheus pod
kubectl exec -it -n monitoring deployment/prometheus-prometheus-server -- /bin/sh

# Access Uptime Kuma pod
kubectl exec -it -n monitoring deployment/uptimekuma -- /bin/sh
```

#### 2. Log Analysis

```bash
# View Grafana logs
kubectl logs -n monitoring deployment/grafana --tail=100

# View Prometheus logs
kubectl logs -n monitoring deployment/prometheus-prometheus-server --tail=100

# View Uptime Kuma logs
kubectl logs -n monitoring deployment/uptimekuma --tail=100
```

#### 3. Port Forwarding for Local Testing

```bash
# Forward Grafana locally
kubectl port-forward -n monitoring service/grafana 3000:3000

# Forward Prometheus locally
kubectl port-forward -n monitoring service/prometheus-prometheus-server 9090:9090

# Access locally
# Grafana: http://localhost:3000
# Prometheus: http://localhost:9090
```

## Monitoring Stack Maintenance

### Updates and Upgrades

#### 1. Update Helm Releases

```bash
# Update Prometheus
helm upgrade -n monitoring prometheus prometheus-community/prometheus --version 25.x

# Update Grafana
helm upgrade -n monitoring grafana grafana/grafana --version 8.x

# Update Uptime Kuma
helm upgrade -n monitoring uptimekuma bjw-s/app-template --set controllers.uptimekuma.containers.app.image.tag=2
```

#### 2. Update Flux

```bash
# Trigger Flux sync
kubectl apply -k homelab/infra-k8s/apps/monitoring

# Check sync status
kubectl get helmreleases -n monitoring -w
```

### Resource Management

#### 1. Monitoring Storage Usage

```bash
# Check PVC usage
kubectl get pvc -n monitoring -o wide

# Monitor PV capacity
kubectl get pv -o wide | grep monitoring
```

#### 2. Scaling Considerations

```bash
# Scale Prometheus if needed
kubectl scale deployment -n monitoring prometheus-prometheus-server --replicas=2

# Scale Grafana if needed
kubectl scale deployment -n monitoring grafana --replicas=2
```

## Related Documentation

- [K3s Cluster Setup](../README.md)
- [Flux CD Configuration](../flux/README.md)
- [Ingress Configuration](../ingress/README.md)
- [Infrastructure Inventory](../../infra/ansible/inventory/inventory.ini)
- [Proxmox Monitoring](../monitoring.md)

## Version History

- **v1.0** (2026-05-29): Initial documentation covering Grafana, Prometheus, and Uptime Kuma setup

---

*This documentation should be updated whenever changes are made to the monitoring stack configuration.*