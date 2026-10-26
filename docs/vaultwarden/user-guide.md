# Vaultwarden User Guide

## Getting Started

### Access

1. Navigate to `https://vault.conderhome.xyz`
2. Click "Create Account" (if registration enabled) or use existing credentials
3. Set up master password (cannot be recovered if lost!)
4. Enable 2FA immediately: Settings → Two-Factor Authentication

### Browser Extension

1. Install Bitwarden extension for your browser
2. Set server URL: `https://vault.conderhome.xyz`
3. Log in with email + master password
4. Enable biometric unlock (if supported)

### Mobile App

1. Install Bitwarden mobile app
2. Settings → Self-hosted environment → `https://vault.conderhome.xyz`
3. Log in
4. Enable biometric/PIN unlock

## Daily Usage

### Adding Items

- **Login**: Website credentials with username/password/URL/TOTP
- **Card**: Credit/debit card details
- **Identity**: Personal info for form filling
- **Secure Note**: Free-form encrypted text

### Organization

- **Folders**: Personal categorization (Work, Personal, Finance)
- **Collections**: Shared with organizations (team sharing)
- **Tags**: Flexible cross-folder labeling (optional)

### Sharing

1. Create Organization (paid Bitwarden feature, free in Vaultwarden)
2. Create Collections within Organization
3. Invite members via email
4. Assign Collections to members/groups

### Emergency Access

- Set up trusted emergency contacts
- Grant time-delayed vault access
- Configure in Settings → Emergency Access

## Import/Export

### Import

Supported formats: Bitwarden JSON/CSV, LastPass, 1Password, KeePass, Chrome, Firefox, etc.

1. Tools → Import Data
2. Select format
3. Upload file
4. Review and confirm

### Export

1. Tools → Export Vault
2. Choose format: JSON (encrypted), CSV, JSON (unencrypted)
3. **Warning**: Unencrypted export contains all secrets in plaintext

## Advanced Features

### Password Generator

- Per-item or standalone
- Configurable length, character sets
- Passphrase mode (diceware-style)

### Data Breach Report

- Checks HaveIBeenPwned for exposed credentials
- Runs automatically on vault unlock
- Manual check: Reports → Data Breach Report

### Send

- Securely share text/files with expiration
- Password-protected optional
- Access logs available

## CLI Usage

```bash
# Login
bw login --server https://vault.conderhome.xyz

# Unlock (returns session key)
export BW_SESSION=$(bw unlock --raw)

# List items
bw list items --session $BW_SESSION

# Get item
bw get item <id> --session $BW_SESSION

# Sync
bw sync --session $BW_SESSION
```

## Troubleshooting

| Issue | Resolution |
|---|---|
| Can't login | Check master password, verify server URL, check 2FA |
| Sync fails | Check internet, run `bw sync`, verify server status |
| Extension not working | Reinstall, clear cache, verify server URL |
| Forgot master password | **Cannot be recovered** - use emergency access or delete account |

## Mobile-Specific

- **Biometric unlock**: FaceID/TouchID (iOS), Fingerprint/Face (Android)
- **Auto-fill**: Enable in system settings → Bitwarden
- **NFC YubiKey**: Supported on Android
- **Offline access**: Vault cached locally after first sync