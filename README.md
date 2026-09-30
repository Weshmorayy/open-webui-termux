# Open WebUI × OpenRouter — Image Studio on Termux

> A self-hosted AI image-generation workspace running entirely on an Android phone via Termux. All inference is remote through OpenRouter — no local models, no GPU required.

## What This Is

A fully working [Open WebUI](https://github.com/open-webui/open-webui) v0.11.4 installation on Android/Termux that connects to [OpenRouter](https://openrouter.ai) for image generation and editing. You upload a product photo, type `/product-clean`, and the AI returns a clean white-background catalog image — all from your phone.

**Architecture:**
```
Android → Termux → Open WebUI (localhost:8080) → OpenRouter API → Image Model
```

## Requirements

- Android phone with ~4 GB RAM
- [Termux](https://f-droid.org/packages/com.termux/) (from F-Droid, not Play Store)
- Termux:API add-on (for `termux-wake-lock`)
- An [OpenRouter](https://openrouter.ai) account and API key
- Internet connection for inference

## Quick Start (after installation)

```bash
# In a Termux terminal — keep this window open
owui-watchdog
```

Then open your Android browser: **http://127.0.0.1:8080**

---

## Installation Guide

See [`docs/INSTALL.md`](docs/INSTALL.md) for the full step-by-step installation.  
See [`docs/TROUBLESHOOTING.md`](docs/TROUBLESHOOTING.md) for every error encountered and how it was fixed.

---

## Scripts

| Script | Description |
|---|---|
| `owui-watchdog` | **Recommended.** Runs server in foreground and auto-restarts if killed |
| `owui-start` | Start server in background (may get killed by Android OOM) |
| `owui-stop` | Stop the server |
| `owui-status` | Show server status and last 20 log lines |

Copy scripts from `bin/` to `~/bin/` and `chmod +x` them.

---

## Features

- **OpenRouter Pipe** — connects Open WebUI to any OpenRouter model
- **Product Image Preset** — `/product-clean` command strips background, removes overlays, outputs white-background catalog image
- **Slim mode** — chromadb, whisper, vector DB, local models all disabled
- **Wake lock** — prevents Android from killing the server process

---

## Usage — Product Image Editing

1. Open `http://127.0.0.1:8080` in your browser
2. Start a new chat
3. **Disable the `get_current_timestamp` tool** (see [Known Issues](docs/TROUBLESHOOTING.md#openrouter-tool-use-error))
4. Select an image model: `openai/gpt-4o-image`, `google/gemini-2.0-flash-exp:image-generation`, etc.
5. Upload your product photo
6. Type `/product-clean` (auto-fills the full prompt)
7. Send — the model returns a white-background product image

---

## Configuration

The server reads `~/open-webui-data/.env`. Key variables:

```bash
WEBUI_NAME="Image Studio"
USE_SLIM_DOCKER=true          # Disables local AI, chromadb, whisper
ENABLE_OLLAMA_API=false
ENABLE_OPENAI_API=false
GLOBAL_LOG_LEVEL=INFO
```

**Your OpenRouter API key is stored only in the Open WebUI database** (encrypted with `WEBUI_SECRET_KEY`). It is never in any script or environment variable.

---

## License

MIT — do whatever you want with it.
