---
title: CloudBeaver User Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [user-guide, documentation, database, cloudbeaver]
---

# CloudBeaver User Guide

## Introduction

CloudBeaver is a self-hosted, browser-based database administration tool running in the homelab's `tools` namespace. It gives you a single place to browse schemas, run SQL, and manage data across every database in the cluster — the shared `shared-postgres` CNPG cluster, `grafana-postgres`, the standalone MySQL deployment, and app-specific databases like readmeabook's — without installing a desktop SQL client.

## Getting Started

### First Time Setup

1. **Access the Web Interface** at `https://cloudbeaver.conderhome.xyz`
2. **Log in as admin** using the credentials stored in the `cloudbeaver-secrets` Kubernetes Secret (`CB_ADMIN_NAME` / `CB_ADMIN_PASSWORD`) — ask whoever manages cluster secrets if you need access
3. **Register Database Connections** for each database you need to work with (see below)
4. **Create additional users** if other people need access, so you don't have to share the admin login

### Dashboard Overview

When you first log in, you'll see:

- **Connections Panel**: List of registered database connections (left sidebar)
- **SQL Editor**: Main workspace for writing and running queries
- **Navigator Tree**: Expandable schema/table browser for the selected connection
- **Results Grid**: Tabular output area below the SQL editor

## Managing Database Connections

### Adding a Connection

1. Click **"New Connection"** (plug icon) in the connections panel
2. Choose the driver: **PostgreSQL** for shared-postgres/grafana-postgres/app Postgres databases, **MySQL** for the MySQL deployment
3. Enter the in-cluster Service host and port, e.g.:
   - `shared-postgres-rw.<namespace>.svc.cluster.local:5432`
   - `grafana-postgres-rw.<namespace>.svc.cluster.local:5432`
   - `mysql.<namespace>.svc.cluster.local:3306`
4. Enter the database name, username, and password for that database
5. Click **"Test Connection"** to confirm connectivity before saving
6. Save — the connection definition is stored in CloudBeaver's workspace (on its PVC) and will persist across pod restarts

### Connecting to the Homelab Databases

| Database | Engine | Typical Use |
|----------|--------|-------------|
| shared-postgres | PostgreSQL (CNPG) | General-purpose shared cluster for multiple tools |
| grafana-postgres | PostgreSQL (CNPG) | Grafana's own metadata store |
| mysql deployment | MySQL | Apps requiring MySQL |
| App-specific DB (e.g. readmeabook) | PostgreSQL | Dedicated per-app database |

You will need the correct in-cluster Service DNS name, port, database name, and credentials for each — these come from each app's own HelmRelease/Secret, not from CloudBeaver.

### Removing or Editing a Connection

1. Right-click the connection in the panel → **Edit Connection** or **Delete Connection**
2. Deleting a connection only removes it from CloudBeaver's registry — it does **not** affect the underlying database or its data

## Browsing and Querying Data

### Navigator Tree

Expand a connection to browse:

- **Databases/Schemas** → **Tables** → **Columns, Indexes, Constraints**
- **Views**, **Functions**, **Triggers** (where supported by the driver)

Double-click a table to open a data grid with filter and sort controls, or right-click → **View Data**.

### Writing SQL

1. Select the target connection/database in the dropdown
2. Open a **new SQL Editor** tab (or use the one opened by default)
3. Write your query and run it with the "Execute" button or `Ctrl+Enter`
4. Results appear in the grid below; use the grid's built-in filter/sort/export controls as needed

### Editing Data

CloudBeaver's data grid supports inline editing for simple cases:

1. Open a table's data view
2. Double-click a cell to edit
3. Use the **Save** (checkmark) button in the grid toolbar to commit changes
4. Be careful: edits here write directly to the underlying database — there is no draft/preview layer beyond the grid's own pending-changes indicator

### Exporting Results

- Use the grid toolbar's **Export** option to save query results as CSV, or other supported formats
- Useful for pulling a one-off report out of shared-postgres or an app database without a separate script

## User Management

### Admin Access

The bootstrap admin account (from `CB_ADMIN_NAME`/`CB_ADMIN_PASSWORD`) can:

- Create additional local users
- Assign roles/permissions within CloudBeaver
- Manage global driver and connection settings

### Creating Additional Users

1. Log in as admin
2. Go to **Administration → Users**
3. Click **"Add User"**, set a username and password, assign a role
4. New users will need their own database connections registered (or shared connections, depending on CloudBeaver's sharing configuration) before they can query anything

## Settings and Configuration

### Workspace Persistence

All connection definitions, saved SQL scripts, and user accounts live on the `cloudbeaver-data` PersistentVolumeClaim (`/opt/cloudbeaver/workspace`). This means:

- Connections you register survive pod restarts and rescheduling
- There is no external config file to edit manually — everything is managed through the web UI once deployed

### Driver Management

CloudBeaver ships with common JDBC drivers (PostgreSQL, MySQL) pre-bundled. Additional drivers can sometimes be added through **Administration → Driver Manager**, though this homelab deployment has not needed any beyond the defaults.

## Troubleshooting

### Common Issues

#### Can't log in

**Issue**: Admin credentials rejected
**Solution**: Confirm you're using the current value of the `cloudbeaver-secrets` Secret — if it was rotated, the old password will no longer work. Check with:
```bash
kubectl get secret cloudbeaver-secrets -n tools -o yaml
```

#### Connection test fails

**Issue**: "Test Connection" fails for a database
**Solution**: Verify the target Service's in-cluster DNS name, port, and that the database user/password are correct for that specific database (not the CloudBeaver admin credentials — those are unrelated). Also confirm the target pod/Service is actually running:
```bash
kubectl get pods,svc -n <target-namespace>
```

#### Queries hang or time out

**Issue**: Long-running query never returns
**Solution**: Check the target database's own load/locks (e.g. `pg_stat_activity` on Postgres) — CloudBeaver has no independent query timeout beyond what the browser session allows; cancel the query from the SQL editor toolbar if needed.

#### UI is unreachable

**Issue**: `cloudbeaver.conderhome.xyz` doesn't load
**Solution**: Check pod and ingress status:
```bash
kubectl get pods -n tools -l app.kubernetes.io/name=cloudbeaver
kubectl describe pod -n tools -l app.kubernetes.io/name=cloudbeaver
kubectl get ingress -n tools
```

### Getting Help

- **This documentation set**: architecture.md, api-documentation.md, deployment-guide.md in `docs/cloudbeaver/`
- **Upstream docs**: DBeaver/CloudBeaver's own documentation site for UI feature details not specific to this deployment
- **Cluster state**: `kubectl` against the cluster for anything infrastructure-related

## Best Practices

- Register one connection per database, named clearly (e.g. `shared-postgres - prod`, `readmeabook-db`) so you don't run a query against the wrong target
- Avoid using CloudBeaver's inline data editing for bulk changes — use proper migrations/scripts for anything beyond a quick manual fix
- Don't store database passwords only inside CloudBeaver connections — keep them recoverable from each app's own Secret as the source of truth
- Treat CloudBeaver as a convenience/admin tool, not a production application dependency — nothing else in the cluster depends on it being up

---

*This documentation is part of the CloudBeaver homelab deployment docs. For architecture and deployment details, see architecture.md and deployment-guide.md in the same directory.*
