# Nebula Sync API Documentation

## Base URL

No external API. Internal cron job only.

## Pi-hole API Endpoints Used

### Primary (Read)

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/custom.list` | Fetch all custom DNS records |
| `GET` | `/api/info` | Version/info (health check) |

Headers: `Authorization: Bearer <PRIMARY_API_KEY>`

### Secondary (Write)

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/custom.list` | Replace entire custom.list |
| `POST` | `/api/dns/update` | Trigger FTL reload |

Headers: `Authorization: Bearer <SECONDARY_API_KEY>`

## Sync Logic

1. Fetch primary `custom.list` (JSON array of `{domain, ip}`)
2. Filter: Only `*.conderhome.xyz` records (optional filter)
3. Compare with secondary's current `custom.list`
4. Compute diff (add/remove/update)
5. If changed: POST full list to secondary
6. Trigger FTL reload on secondary
7. Log: added/removed/unchanged counts

## Error Handling

- Primary unreachable → Log error, skip run
- Secondary unreachable → Log error, skip run
- Auth failure → Alert (check API keys)
- Diff empty → Log "no changes", skip write
- FTL reload fails → Log warning, manual reload may be needed

## Manual Trigger

```bash
# Via kubectl (creates one-off job)
kubectl create job --from=cronjob/nebula-sync manual-sync-$(date +%s) -n tools

# Check logs
kubectl logs -n tools job/manual-sync-<timestamp>
```