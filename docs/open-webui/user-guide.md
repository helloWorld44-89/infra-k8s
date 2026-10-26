# Open WebUI User Guide

## Access

Navigate to `https://ai.conderhome.xyz`

First visit: create admin account (email + password).

## Models

### Adding Models

1. Settings (gear icon) → Models
2. **Ollama** (local): Enter Ollama URL (e.g., `http://ollama:11434`), click "Pull" next to model
2. **OpenAI**: Enter API key, model ID (e.g., `gpt-4o`)
3. **OpenAI-compatible**: Base URL + API key + model ID

### Managing Models

- **Pull**: Download from Ollama registry
- **Delete**: Remove local copy (frees disk space)
- **Set Default**: Star model to make default for new chats

## Chats

### Starting a Chat

1. Click "New Chat"
2. Select model (top dropdown)
3. Optional: System prompt (gear icon in chat)
4. Type message, press Enter

### Features

- **Streaming**: Token-by-token response (default)
- **Code blocks**: Syntax highlighting, copy button
- **Math**: LaTeX rendering (`$...$` inline, `$$...$$` block)
- **Regenerate**: Circular arrow to redo last response
- **Branch**: Fork conversation at any message
- **Search**: Ctrl/Cmd + K to search chat history

### File Attachments

- Drag & drop or click 📎
- Supported: PDF, TXT, MD, CSV, JSON, images
- Auto-added to RAG context if knowledge base selected
- Images: Vision models can analyze

### RAG (Retrieval-Augmented Generation)

1. Create Knowledge Base: Settings → Knowledge → New
2. Upload documents (PDF, TXT, MD, etc.)
2. In chat: Select knowledge base (book icon)
3. Ask questions - relevant chunks auto-injected

### Prompts

- Save reusable prompts: Chat → Save as Prompt
- Use variables: `{{variable}}` → prompted at runtime
- Share: Export/import prompt JSON

### Functions / Tools

- Register custom functions (OpenAI function calling format)
- Model can call your functions during chat
- Define in Settings → Functions

## Settings

### General

- Theme: Light/Dark/System
- Language
- Streaming on/off
- Default model

### Account

- Change password
- Generate API keys (for programmatic access)
- Two-factor auth (if enabled)
- Export/import data

### Admin (first user)

- User management
- Global settings (default model, ollama URL, etc.)
- Branding (title, logo)
- Disable registration
- OIDC/SAML config (if configured)

## Keyboard Shortcuts

- `Ctrl/Cmd + Enter`: Send message
- `Ctrl/Cmd + Shift + Enter`: New line
- `Ctrl/Cmd + K`: Search chats
- `Ctrl/Cmd + N`: New chat
- `Ctrl/Cmd + /`: Toggle sidebar
- `↑/↓`: Navigate history (in input)

## Mobile

- Responsive design works in mobile browser
- Add to Home Screen for app-like experience
- Touch-optimized

## Troubleshooting

| Issue | Fix |
|---|---|
| Model not responding | Check Ollama/OpenAI connectivity, model loaded |
| Slow responses | Reduce context, use smaller model |
| RAG not working | Re-index knowledge base, check vector DB |
| Lost admin | Delete `webui.db`, restart (recreates admin on first signup) |