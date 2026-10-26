# KeyShift User Guide

## Access

Navigate to `https://keyshift.conderhome.xyz`

Login with configured admin credentials.

## Dashboard

Shows current SSIDs, their status, and next rotation schedule.

## SSID Management

### Adding SSID

1. Click "Add SSID"
2. Enter:
   - **Name**: Friendly name
   - **SSID**: Broadcast name
   - **Password**: WPA2/WPA3 passphrase
   - **Band**: 2.4GHz / 5GHz / Both
3. Save

### Rotating Passwords

- Manual: Click "Rotate" on any SSID
- Scheduled: Configure cron-like schedule
- Generates new random passphrase
- Updates router via API (if configured)
- Generates new QR codes

### QR Codes

- Guest network QR codes auto-generated
- Display on dashboard or print
- Includes SSID + password + security type

## Scheduling

Cron-style syntax for automatic rotations:
```
* * * * *  (minute hour day month weekday)
```

Example daily at 3 AM: `0 3 * * *`

## Audit Logs

View rotation history:
- Timestamp
- SSID affected
- Old password (masked)
- Trigger (manual/scheduled/api)
- Success/failure

## Settings

- **Timezone**: America/Chicago
- **Rotation password length**: Default 16 chars
- **Router API**: Configure if auto-rotate via router API
- **Notifications**: Email/webhook on rotation (if configured)

## Troubleshooting

| Issue | Fix |
|---|---|
| Can't login | Check credentials, clear cookies |
| Rotation fails | Verify router API credentials, check logs |
| QR not generating | Check SSID exists and has password |
| Rate limited (429) | Wait, or use `/healthz` for monitoring |