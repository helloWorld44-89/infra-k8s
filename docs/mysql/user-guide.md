---
title: MySQL User Guide
date: 2026-10-23
author: Hermes Docs Agent
tags: [user-guide, documentation, database, mysql]
---

# MySQL User Guide

## Introduction

This MySQL instance is a shared database service running in the `tools` namespace of the homelab k3s cluster. It has **no web interface of its own** — it is a plain MySQL 8 server. This guide covers the two supported ways to connect and work with it: the **CloudBeaver** web admin UI, and the **`mysql` CLI** (directly in-cluster or via port-forward from your workstation).

## Getting Started

### What You're Connecting To

- **Service**: `mysql.tools.svc.cluster.local:3306`
- **Default database**: `my_database`
- **Credentials**: stored in the `mysql-secrets` Kubernetes Secret (`tools` namespace) — ask a cluster admin, or if you have `kubectl` access see "Retrieving Credentials" below

### Retrieving Credentials

If you have `kubectl` access to the cluster:

```bash
kubectl get secret mysql-secrets -n tools -o jsonpath='{.data.MYSQL_USER}' | base64 -d; echo
kubectl get secret mysql-secrets -n tools -o jsonpath='{.data.MYSQL_PASSWORD}' | base64 -d; echo
```

Root credentials (admin use only):

```bash
kubectl get secret mysql-secrets -n tools -o jsonpath='{.data.MYSQL_ROOT_PASSWORD}' | base64 -d; echo
```

## Method 1: CloudBeaver (Recommended for Browsing/Querying)

CloudBeaver is a web-based database client already deployed alongside MySQL in the `tools` namespace.

### Accessing CloudBeaver

1. Open `https://cloudbeaver.conderhome.xyz` in your browser
2. Log in with the CloudBeaver admin credentials (from `cloudbeaver-secrets`, set up separately from the MySQL credentials)

### Adding the MySQL Connection (first time only)

1. Click **"New Connection"**
2. Choose **MySQL** as the driver
3. Enter connection details:
   - **Host**: `mysql.tools.svc.cluster.local`
   - **Port**: `3306`
   - **Database**: `my_database`
   - **Username / Password**: the `MYSQL_USER` / `MYSQL_PASSWORD` values retrieved above
4. Test the connection, then save

### Using CloudBeaver

- **SQL Editor**: Run ad-hoc `SELECT`/`INSERT`/`UPDATE`/`DDL` statements against `my_database`
- **Schema Navigator**: Browse tables, columns, indexes, and views in the left-hand tree
- **Data Viewer**: Browse/edit table rows in a spreadsheet-like grid
- **Export**: Export query results or whole tables to CSV/JSON/SQL

This is the recommended path for day-to-day inspection, one-off queries, and manual data fixes — no local tooling required.

## Method 2: mysql CLI

For scripting, migrations, or when you need something CloudBeaver doesn't offer (e.g. `mysqldump`).

### From Inside the Cluster

Run a temporary client pod or `kubectl exec` into the MySQL pod itself:

```bash
# Exec into the running mysql pod directly
kubectl exec -it -n tools deploy/mysql -- mysql -u<MYSQL_USER> -p my_database
```

### From Outside the Cluster (port-forward)

```bash
# Forward the service to your local machine
kubectl port-forward -n tools svc/mysql 3306:3306

# In another terminal
mysql -h 127.0.0.1 -P 3306 -u <MYSQL_USER> -p my_database
```

### Common CLI Tasks

**List databases:**
```sql
SHOW DATABASES;
```

**List tables in the default database:**
```sql
USE my_database;
SHOW TABLES;
```

**Describe a table:**
```sql
DESCRIBE <table_name>;
```

**Dump the database (backup):**
```bash
kubectl exec -n tools deploy/mysql -- \
  mysqldump -uroot -p"$MYSQL_ROOT_PASSWORD" my_database > my_database-backup.sql
```

**Restore from a dump:**
```bash
kubectl exec -i -n tools deploy/mysql -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" my_database < my_database-backup.sql
```

## Who Should Use This Database

This is a **shared, general-purpose MySQL instance** for the `tools` namespace rather than a dedicated backend for one specific app. As of this writing, no HelmRelease in the repository wires an application to it via `DB_HOST`/`MYSQL_HOST` — it is primarily reached manually through CloudBeaver or the `mysql` CLI by cluster administrators for ad-hoc data work, testing, or as a ready-made MySQL target for a future app. If you deploy a new app that needs MySQL, point it at `mysql.tools.svc.cluster.local:3306` / `my_database` and update the architecture doc with the new consumer.

## Troubleshooting

### Common Issues

**Issue**: Can't connect, `ERROR 2002 (HY000)`
**Solution**: Confirm you're either running inside the cluster (and the service name resolves) or have an active `kubectl port-forward` running; check the pod is `Running`: `kubectl get pods -n tools -l app.kubernetes.io/name=mysql`

**Issue**: `ERROR 1045 (28000): Access denied for user`
**Solution**: Double-check the username/password pulled from `mysql-secrets` — re-run the `kubectl get secret` commands above, values may have been rotated

**Issue**: `ERROR 1049 (42000): Unknown database`
**Solution**: The only database created by default is `my_database`; confirm you didn't mistype the schema name

**Issue**: CloudBeaver connection test fails but `mysql` CLI works
**Solution**: Check CloudBeaver's pod logs (`kubectl logs -n tools deploy/cloudbeaver`) and confirm the driver is set to MySQL (not MariaDB) with the correct port

### Getting Help

- **Architecture**: See `docs/mysql/architecture.md` for how this service fits into the cluster
- **Deployment**: See `docs/mysql/deployment-guide.md` for how it's provisioned and how to change its configuration
- **Logs**: `kubectl logs -n tools deploy/mysql`
- **Pod status**: `kubectl describe pod -n tools -l app.kubernetes.io/name=mysql`

## Best Practices

- Use the scoped `MYSQL_USER` application account for routine work; reserve `root` for administrative tasks (grants, backups, restores)
- Take a `mysqldump` backup before any destructive schema change
- Don't store large binary blobs in this database — the PVC is sized at 1Gi and is not intended for bulk media storage
- If you start consuming this database from a new application, record that in `docs/mysql/architecture.md` so future readers know who depends on it

---

*This guide covers connecting to and working with the shared `mysql` service in the `tools` namespace. There is no end-user application UI for this service — administration happens through CloudBeaver or the `mysql` CLI.*
