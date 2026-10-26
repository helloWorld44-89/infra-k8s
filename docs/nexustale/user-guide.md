# NexusTale User Guide

## Access

Navigate to `https://nexustale.conderhome.xyz`

Login with email/password. First user registers automatically.

## Dashboard

- **Recent notes**: Quick access
- **Collections**: Organized groups
- **Tags**: Filter by tag
- **Search bar**: Global search (semantic + keyword)

## Notes

### Creating Notes

1. Click "New Note" (+)
2. Title + content (Markdown supported)
3. Optional: Assign to collection
4. Tags: Type `#tag` or use tag picker
4. Auto-saves as you type

### Editor

- **Toolbar**: Bold, italic, code, headers, lists, links, images
- **Shortcuts**: `Ctrl+B` bold, `Ctrl+I` italic, `Ctrl+K` link
- **Code blocks**: Triple backticks + language
- **Math**: `$inline$` and `$$block$$`
- **Mermaid**: ```mermaid diagrams

### Organization

- **Collections**: Folders for grouping (nestable)
- **Tags**: Cross-cutting labels (`#project`, `#reference`)
- **Favorites**: Star important notes
- **Archive**: Hide without deleting

### Linking

- `[[note-title]]` → WikiLink to other notes
- `#tag` → Tag filter
- `@collection` → Collection filter
- Auto-complete on `[[` and `#`

### Attachments

- Drag & drop images/files
- Stored in backend, linked in note
- Max size: 50MB per file

## Search

### Global Search

- Header search bar (or `/`)
- Full-text + semantic (vector)
- Filters: `collection:`, `tag:`, `date:`

### Semantic Search

- Natural language queries
- "Notes about project planning"
- Uses pgvector embeddings

### Hybrid Search

- Combines keyword + vector
- Best of both worlds

## Collections

- Create: Sidebar → New Collection
- Nest: Drag onto another collection
- Color: Right-click → Color
- Share: (Future) Share with other users

## Tags

- Inline: `#tag-name` in note
- Manage: Tags page (merge, rename, delete)
- Filter: Click tag anywhere to filter

## Settings

### Account

- Profile: Name, avatar, email
- Password: Change
- 2FA: TOTP (if enabled)

### Editor

- Theme: Light/Dark/Auto
- Font size
- Vim mode (experimental)
- Auto-save interval

### Sync / Backup

- Export: JSON (all notes + metadata)
- Import: JSON (merge or replace)
- Auto-backup: (Future) scheduled exports

## Keyboard Shortcuts

| Shortcut | Action |
|---|---|
| `Ctrl/Cmd + N` | New note |
| `Ctrl/Cmd + S` | Save (manual) |
| `Ctrl/Cmd + /` | Toggle sidebar |
| `Ctrl/Cmd + K` | Command palette |
| `Ctrl/Cmd + P` | Quick open |
| `Ctrl/Cmd + Shift + F` | Global search |
| `[[` | WikiLink autocomplete |
| `#` | Tag autocomplete |

## Mobile

- Responsive: Works in mobile browser
- PWA: Add to Home Screen
- Touch-optimized editor
- Offline: (Future) Service worker cache

## Troubleshooting

| Issue | Fix |
|---|---|
| Slow load | Check network, clear browser cache |
| Embeddings not working | Check API `/health`, pgvector extension |
| Sync issues | Check backend logs, Redis connectivity |
| Lost data | Check `deleted_at` (soft delete), restore from archive |