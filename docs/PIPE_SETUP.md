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

## Disabling the Tool Use Conflict

Image models don't support the `tools` parameter. Open WebUI enables `get_current_timestamp` by default, which causes this error:

```
No endpoints found that support tool use. Try disabling "get_current_timestamp".
```

**Fix per chat:**
- Click the Tools icon in the chat input bar → toggle OFF `get_current_timestamp`

**Fix globally (admin):**
- Admin Panel → Tools → `get_current_timestamp` → toggle global default OFF
