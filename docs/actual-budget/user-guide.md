# Actual Budget User Guide

## Access

Navigate to `https://budget.conderhome.xyz`

Create account or login with existing credentials.

## First-Time Setup

1. **Create Account**: Email + strong password (used to derive encryption key)
2. **Create Budget**: Click "New Budget" → name it (e.g., "Personal", "Family")
3. **Set Up Accounts**: Add checking, savings, credit cards, loans, investments
4. **Configure Categories**: Income, expenses, transfers (use defaults or customize)
5. **Set Up Sync** (optional): File-based sync works automatically

## Core Concepts

### Envelope Budgeting

- Every dollar assigned a job
- Categories = envelopes
- Rollovers: unspent carries to next month
- Credit cards: budget for payment, not individual transactions

### Reconciliation

- Match transactions to bank statements
- Green = cleared, Yellow = uncleared, Red = needs review
- Monthly reconciliation recommended

### Reports

- **Net Worth**: Assets - liabilities over time
- **Income vs Expense**: Monthly comparison
- **Spending by Category**: Visual breakdown
- **Cash Flow**: In vs out per month

## Sync

- **Automatic**: File-based sync runs in background
- **Conflict Resolution**: Last-write-wins (manual merge if needed)
- **Mobile**: Install Actual app, enter server URL, login
- **Offline**: Full offline support, syncs when online

## Import/Export

### Import

Supported: CSV, OFX, QFX, QIF, JSON (Actual format)

1. Settings → Import
2. Select file
3. Map columns
4. Review and confirm

### Export

- Settings → Export → JSON (encrypted) or CSV
- **Warning**: Encrypted export requires password to decrypt

## Advanced

### Rules

Auto-categorize transactions:
- Payee contains "Starbucks" → Category: Coffee
- Amount = $1200 → Category: Rent

### Recurring Transactions

- Set up bills, subscriptions, income
- Auto-create on schedule
- Review before posting

### Multiple Budgets

- Separate budgets for different purposes
- Switch via dropdown
- Each budget fully independent

## Keyboard Shortcuts

- `n`: New transaction
- `e`: Edit selected
- `r`: Reconcile
- `/`: Search
- `?`: Show all shortcuts

## Mobile

- Install Actual app (iOS/Android)
- Settings → Server URL: `https://budget.conderhome.xyz`
- Login
- Full offline support
- Biometric unlock (if enabled)

## Troubleshooting

| Issue | Fix |
|---|---|
| Can't sync | Check server URL, internet, restart app |
| Forgot password | **Cannot reset** - encryption key derived from password |
| Data missing | Check sync status, verify correct budget selected |
| Mobile won't connect | Verify server URL uses HTTPS, check cert |