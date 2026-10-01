# OpenRouter Pipe Setup

Instructions for configuring the OpenRouter pipe inside Open WebUI.

---

## What the Pipe Does

The [Open WebUI OpenRouter Pipe](https://github.com/rbb-dev/Open-WebUI-OpenRouter-pipe) (v2.7.3) acts as a proxy between Open WebUI and OpenRouter's API. It:

- Fetches the list of available models from OpenRouter and displays them in the model selector
- Routes your chat/image requests to OpenRouter
- Stores your API key securely inside Open WebUI's encrypted database (never in config files)

---

## Installation (Already Done in This Setup)

The pipe code is bundled at `~/open-webui-data/open_webui_openrouter_pipe_bundled.py` and pre-seeded into the database. You only need to add your API key.

### Manual DB Seed (if needed)

```python
import sqlite3, json, datetime

PIPE_PATH = "/data/data/com.termux/files/home/open-webui-data/open_webui_openrouter_pipe_bundled.py"
DB_PATH = "/data/data/com.termux/files/home/open-webui-data/webui.db"

with open(PIPE_PATH) as f:
    code = f.read()

now = int(datetime.datetime.now().timestamp())

conn = sqlite3.connect(DB_PATH)
conn.execute("""
    INSERT OR REPLACE INTO function
    (id, user_id, name, type, content, meta, is_active, is_global, updated_at, created_at)
    VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
""", (
    "openrouter_pipe",
    "admin",
    "OpenRouter Pipe",
    "pipe",
    code,
    json.dumps({"description": "OpenRouter image & chat models", "manifest": {}}),
    1,  # is_active
    1,  # is_global
    now,
    now,
))
conn.commit()
conn.close()
print("Pipe seeded.")
```

**Important:** The pipe's frontmatter `requirements:` line must be empty or the server will try to `pip install imageio numpy` at startup, which fails on Android:

```python
# In the pipe source, ensure this line reads:
# requirements:
# (not: requirements: aiohttp, imageio, numpy, ...)
```

---

## Adding Your API Key

1. Open **http://127.0.0.1:8080** → Admin Panel → Functions
2. Find **OpenRouter Pipe** — make sure the toggle is ON
3. Click the **⚙️ gear icon** next to it
4. Enter your [OpenRouter API key](https://openrouter.ai/keys)
5. Click **Save**

The key is stored encrypted inside `webui.db` using your `WEBUI_SECRET_KEY`. It is never written to any file on disk in plaintext.

---

## Selecting Models

After adding your API key:

1. Start a new chat
2. Click the **model selector** dropdown at the top
3. Search for image models:
   - `openai/gpt-4o-image` — GPT-4o with image generation
   - `google/gemini-2.0-flash-exp:image-generation` — Gemini image generation
   - `openai/gpt-5` — GPT-5 (if available on your plan)

---

## How the Model List Is Built

The pipe holds **no hardcoded model list** — it fetches OpenRouter's catalog
live, so new models appear without editing anything.

Verified coverage against the live API (measured 2026-10-01; these totals drift
as OpenRouter adds models — re-check rather than trusting the numbers):

| | Live OpenRouter | Registered in Open WebUI |
|---|---|---|
| Chat models | 464 | 458 (the other 5 are base-vs-`:batch` variants) |
| Image models | 55 | **55 — all of them** |
| Audio-output chat models | 4 | **4** (`lyria-3-pro`, `lyria-3-clip`, `gpt-audio`, `gpt-audio-mini`) |
| Video | separate endpoints | registered as dedicated filters |

Video models (Veo, Sora, Kling, Wan) and the image-gen options (Recraft,
Gemini, Sourceful, Grok Imagine) appear as **separate filter functions** in the
Admin Panel, not as chat entries — that is how Open WebUI models non-chat media.

### ID format in the database

Open WebUI sanitises model ids, so they look different in the DB than on the
API. `stealth/space-bunny-alpha` is stored as:

```
openrouter_pipe.stealth.space-bunny-alpha
```

`/` and `:` become `.`. Compare ids with that in mind when querying `webui.db`.

### Refresh cadence

The pipe caches the catalog in memory and re-fetches when stale:

```
cache_seconds = MODEL_CATALOG_REFRESH_SECONDS   # default 3600 = 1 hour
next_refresh  = _last_fetch + cache_seconds
```

No restart is needed, and no external script — unlike the DeepSeek Harness and
FreeLLMAPI setups, which refresh on boot (see `Weshmorayy/ai-system-termux`).

### Valves that can hide models

If a model is missing, check these in **Admin Panel → Functions → OpenRouter
Pipe → gear icon**. Current values in this setup:

| Valve | Value | Effect |
|---|---|---|
| `MODEL_ID` | `auto` | imports every Responses-capable model |
| `FREE_MODEL_FILTER` | `all` | `only` hides all paid models; `exclude` hides free ones |
| `TOOL_CALLING_FILTER` | `all` | `only` hides non-tool models |
| `ZDR_MODELS_ONLY` | `false` | `true` hides models with no zero-retention endpoint |
| `NEW_MODEL_ACCESS_CONTROL` | `admins` | `public` grants all users read access to newly added models |

> **`NEW_MODEL_ACCESS_CONTROL` is the usual culprit.** With `admins`, a model
> can be registered but invisible to non-admin users. Set `public` if teammates
> cannot see models you know exist.

### Why free / stealth models DO appear

The pipe defines free as *"all pricing values sum to 0"*:

```python
# bundled pipe, is_free_model(), line 19900
pricing = OpenRouterModelRegistry.spec(model_norm_id).get("pricing") or {}
total, numeric_count = sum_pricing_values(pricing)
if numeric_count <= 0:
    return False
return total == Decimal(0)
```

This is a **price** check, not a `:free` suffix check — which is why
`stealth/space-bunny-alpha` (no `:free` suffix, priced at 0) is correctly
treated as free. With `FREE_MODEL_FILTER=all` nothing is filtered anyway.

---

## Disabling the Tool Use Conflict

Image models don't support the `tools` parameter. Open WebUI enables `get_current_timestamp` by default, which causes this error:

```
No endpoints found that support tool use. Try disabling "get_current_timestamp".
```

**Fix per chat:**
- Click the Tools icon in the chat input bar → toggle OFF `get_current_timestamp`

**Fix globally (admin):**
- Admin Panel → Tools → `get_current_timestamp` → toggle global default OFF
