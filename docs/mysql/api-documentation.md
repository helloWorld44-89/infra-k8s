---
title: MySQL API Documentation
date: 2026-10-23
author: Hermes Docs Agent
tags: [api, documentation, database, mysql]
---

# MySQL API Documentation

## Overview

MySQL in this cluster has no REST/HTTP API — it speaks only the **MySQL wire protocol** (binary TCP protocol, port 3306). This document covers the connection parameters, protocol-level behavior, and client libraries/tools needed to talk to it programmatically. If you are looking for an HTTP API to curl, there isn't one; use a MySQL client library or the `mysql` CLI instead.

**Protocol**: MySQL Client/Server Protocol (binary, over TCP)

**In-cluster endpoint**: `mysql.tools.svc.cluster.local:3306`

**Port-forwarded local endpoint** (for access from outside the cluster): `127.0.0.1:3306` (or any local port you choose)

**Default schema**: `my_database`

**Authentication**: MySQL native user/password auth (`mysql_native_password` / `caching_sha2_password` depending on client), resolved against the `mysql.user` grant tables inside the server. No external OIDC/SSO/API-key layer — credentials are the username/password pair provisioned from the `mysql-secrets` Kubernetes Secret.

## Connection Details

| Parameter | Value |
|-----------|-------|
| Host (in-cluster) | `mysql.tools.svc.cluster.local` |
| Host (via port-forward) | `127.0.0.1` (or your forwarded host) |
| Port | `3306` |
| Protocol | MySQL wire protocol (TCP) |
| Default database | `my_database` |
| Root user | `root` (password from `mysql-secrets` key `MYSQL_ROOT_PASSWORD`) |
| App user | value of `mysql-secrets` key `MYSQL_USER` (password: `MYSQL_PASSWORD`) |
| TLS | Not configured — connect in-cluster only, or over a trusted tunnel/port-forward |

Credentials live in the `mysql-secrets` Secret in the `tools` namespace. Retrieve them (for authorized cluster admins only):

```bash
kubectl get secret mysql-secrets -n tools -o jsonpath='{.data.MYSQL_USER}' | base64 -d
kubectl get secret mysql-secrets -n tools -o jsonpath='{.data.MYSQL_PASSWORD}' | base64 -d
kubectl get secret mysql-secrets -n tools -o jsonpath='{.data.MYSQL_ROOT_PASSWORD}' | base64 -d
```

## Connecting Programmatically

### From a Pod in the Cluster

Any workload in the k3s cluster can resolve the service by its cluster DNS name and connect directly on port 3306 — no port-forward or ingress needed:

```
host:     mysql.tools.svc.cluster.local
port:     3306
database: my_database
```

### mysql CLI

```bash
mysql -h mysql.tools.svc.cluster.local -P 3306 -u <MYSQL_USER> -p my_database
```

### Python (PyMySQL / mysql-connector-python)

```python
import pymysql

conn = pymysql.connect(
    host="mysql.tools.svc.cluster.local",
    port=3306,
    user="<MYSQL_USER>",
    password="<MYSQL_PASSWORD>",
    database="my_database",
)

with conn.cursor() as cursor:
    cursor.execute("SELECT VERSION();")
    print(cursor.fetchone())

conn.close()
```

### Node.js (mysql2)

```javascript
const mysql = require('mysql2/promise');

const connection = await mysql.createConnection({
  host: 'mysql.tools.svc.cluster.local',
  port: 3306,
  user: process.env.MYSQL_USER,
  password: process.env.MYSQL_PASSWORD,
  database: 'my_database',
});

const [rows] = await connection.execute('SELECT VERSION();');
console.log(rows);

await connection.end();
```

### Connection String Formats

```
# Generic DSN
mysql://<user>:<password>@mysql.tools.svc.cluster.local:3306/my_database

# SQLAlchemy
mysql+pymysql://<user>:<password>@mysql.tools.svc.cluster.local:3306/my_database

# JDBC
jdbc:mysql://mysql.tools.svc.cluster.local:3306/my_database
```

## Protocol-Level Notes

- **Handshake**: Standard MySQL 8 handshake (`caching_sha2_password` by default; clients without SHA-256 RSA support may need `mysql_native_password` enabled server-side or `--ssl-mode=DISABLED` / `allowPublicKeyRetrieval=true` depending on client).
- **Charset**: Server defaults to `mysql:8` image defaults (`utf8mb4`) unless overridden.
- **Max connections / timeouts**: Unmodified `mysql:8` defaults — no custom `my.cnf` is mounted by this HelmRelease.
- **Transactions**: Standard InnoDB ACID transactions; `my_database` uses InnoDB as the default storage engine.

## Access via CloudBeaver (No Direct API Needed)

For ad-hoc queries, schema browsing, or data management, prefer **CloudBeaver** (`https://cloudbeaver.conderhome.xyz`) over writing a client — see the User Guide for setup. CloudBeaver itself exposes its own internal HTTP API for its UI, but that is unrelated to and does not substitute for a direct MySQL connection; it is simply a GUI client that opens a MySQL wire-protocol connection on your behalf.

## Error Handling

MySQL communicates errors via native protocol error packets, surfaced by client libraries as exceptions/error objects containing:

| Field | Description |
|-------|-------------|
| `errno` | MySQL numeric error code (e.g. `1045` Access denied, `1049` Unknown database, `2002`/`2003` Can't connect) |
| `sqlstate` | ANSI SQL state code |
| `message` | Human-readable error text |

Common codes relevant to this deployment:

| Error | Meaning | Likely Cause |
|-------|---------|--------------|
| `2002` | Can't connect to local MySQL server | Wrong host/port, or not running in-cluster/port-forward not active |
| `1045` | Access denied for user | Wrong username/password from `mysql-secrets` |
| `1049` | Unknown database | Database name other than `my_database` |
| `1129` | Host blocked | Too many failed connection attempts (run `FLUSH HOSTS;` as root to clear) |

## Rate Limiting

No application-level rate limiting exists for raw MySQL connections. Standard MySQL server limits apply (`max_connections`, default ~151 for `mysql:8`). Since this is a single-replica instance with limited resources, avoid opening large unbounded connection pools from client applications.

---

*This document covers the MySQL wire-protocol interface for the shared `mysql` service in the `tools` namespace. There is no REST/HTTP API for this database; use a MySQL client library, the `mysql` CLI, or CloudBeaver.*
