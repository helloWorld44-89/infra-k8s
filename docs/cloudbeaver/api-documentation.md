---
title: CloudBeaver API Documentation
date: 2026-10-23
author: Hermes Docs Agent
tags: [api, documentation, database, cloudbeaver]
---

# CloudBeaver API Documentation

## Overview

CloudBeaver is primarily a **web UI tool**, not an application with a public, documented REST API meant for third-party integration. Unlike AudioFile's stable, versioned `/api/*` surface, CloudBeaver's HTTP endpoints are the internal API that its own GWT web client uses to talk to the backend — they exist to support the browser UI, are subject to change between CloudBeaver releases, and are not officially published for external consumption. They are documented here for operational/debugging purposes only (e.g. scripting a health check or a one-off SQL execution against this deployment), not as a stable integration contract.

**Base URL**: `https://cloudbeaver.conderhome.xyz` (in-cluster: `http://cloudbeaver.tools.svc.cluster.local:8978`)

**Authentication**:
- Session-cookie based, established via the internal login call
- Admin bootstrap credentials come from the `cloudbeaver-secrets` Kubernetes Secret (`CB_ADMIN_NAME` / `CB_ADMIN_PASSWORD`)

**Content-Type**: `application/json` (CloudBeaver's internal API uses a GraphQL-style single-endpoint RPC pattern rather than separate REST resources)

## Internal API Shape

CloudBeaver CE exposes most of its functionality through a single GraphQL endpoint rather than many individual REST routes. There is no OpenAPI/Swagger spec for this; the schema is internal to the product and can shift between versions (this deployment runs `25.0.1`).

### GraphQL Endpoint

```http
POST /api/gql
```

All UI actions — listing connections, opening a session, running SQL, fetching results — are GraphQL operations against this single endpoint.

**Example: Server Config Query**
```bash
curl -s -X POST https://cloudbeaver.conderhome.xyz/api/gql \
  -H "Content-Type: application/json" \
  --cookie-jar cookies.txt --cookie cookies.txt \
  -d '{"query": "query { serverConfig { name version } }"}'
```

**Typical Response Shape**
```json
{
  "data": {
    "serverConfig": {
      "name": "CloudBeaver",
      "version": "25.0.1"
    }
  }
}
```

### Session Bootstrap

```http
GET /api/sql/status
```

Used by the client to confirm the backend is up and a session is live. Returns a plain status payload; not part of a stable contract.

### Authentication (Internal)

The web client performs a GraphQL `authLogin` mutation with the admin (or a created user's) credentials to establish a session cookie. There is no separate token/API-key mechanism in CloudBeaver CE — session cookies are the only auth artifact.

```bash
# Conceptual login flow (fields vary by CloudBeaver version)
curl -s -X POST https://cloudbeaver.conderhome.xyz/api/gql \
  -H "Content-Type: application/json" \
  --cookie-jar cookies.txt \
  -d '{
    "query": "mutation Login($credentials: Object!) { authLogin(provider: \"local\", credentials: $credentials) { authId } }",
    "variables": { "credentials": { "user": "***", "password": "***" } }
  }'
```

Treat credentials here exactly as sensitive as the admin password itself — never hardcode them in scripts or commit them to the repo.

### SQL Execution (Internal, via GraphQL)

SQL execution is performed via `sqlExecuteQuery`/`sqlResultClose`-style GraphQL mutations scoped to a previously opened connection and context. These require a valid session cookie and an existing database connection ID already registered in CloudBeaver's workspace — there is no simple "run this SQL against this host/port" stateless endpoint.

```bash
# Conceptual shape only — exact mutation fields depend on CloudBeaver version
curl -s -X POST https://cloudbeaver.conderhome.xyz/api/gql \
  -H "Content-Type: application/json" \
  --cookie cookies.txt \
  -d '{
    "query": "mutation Execute($connectionId: ID!, $sql: String!) { sqlExecuteQuery(connectionId: $connectionId, query: $sql) { resultId } }",
    "variables": { "connectionId": "shared-postgres", "sql": "SELECT 1" }
  }'
```

### Health / Liveness

CloudBeaver does not ship a dedicated `/health` endpoint comparable to AudioFile's. For this deployment, liveness is assessed at the Kubernetes level instead:

```bash
kubectl get pods -n tools -l app.kubernetes.io/name=cloudbeaver
kubectl describe pod -n tools -l app.kubernetes.io/name=cloudbeaver
```

A simple unauthenticated reachability check can be done against the login page:

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://cloudbeaver.conderhome.xyz
```

## Error Handling

GraphQL responses use the standard GraphQL error envelope rather than HTTP status codes for application-level errors:

```json
{
  "errors": [
    {
      "message": "Connection 'shared-postgres' not found",
      "path": ["sqlExecuteQuery"]
    }
  ],
  "data": null
}
```

HTTP-level errors (auth failures, bad routes) still use standard status codes (401, 403, 404, 500).

## Rate Limiting

CloudBeaver CE does not implement configurable API rate limiting. There is no equivalent to AudioFile's per-endpoint request budgets; concurrency is instead bounded naturally by the single pod/single JDBC connection pool per registered database connection.

## Recommendation for Automation

Because the internal GraphQL schema is undocumented and version-dependent, any automation against this CloudBeaver instance should:
1. Treat it as **best-effort/internal-only**, not a stable integration surface
2. Prefer connecting directly to the underlying databases (shared-postgres, grafana-postgres, mysql, app DBs) with a proper client/driver for scripted or programmatic work
3. Reserve CloudBeaver for interactive, human-driven SQL browsing and admin tasks through the web UI

## SDK Libraries

None. DBeaver/CloudBeaver does not publish an official client SDK for its internal API. For programmatic database access in this homelab, connect directly to the target database (e.g. `psql`, `mysql` CLI, or a language-native driver) rather than proxying through CloudBeaver.

---

*This API documentation is part of the CloudBeaver homelab deployment docs. CloudBeaver's internal API is undocumented upstream and may change across versions — treat all endpoint shapes above as observed/best-effort, not a guaranteed contract.*
