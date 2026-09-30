# Product Image Preset — `/product-clean`

The `/product-clean` command fills the chat input with a concise image-editing instruction. You upload a product photo, type `/product-clean`, and send — the model returns a clean catalog image.

---

## What It Does

Three things, in one shot:

1. **Clean up** the image (artifacts, noise, compression, dirt)
2. **Center the product** on a pure white `#FFFFFF` background
3. **Remove all text and graphics** — overlays, stickers, watermarks, banners, logos printed on the background — anything that is NOT physically part of the product

**Nothing on the product itself changes** — shape, colors, materials, labels, packaging details are preserved exactly.

---

## The Prompt (exact text sent to the model)

```
Clean up the image, center the product, and place it on a pure white #FFFFFF background.
Remove all text, graphics, logos, stickers, watermarks, and overlays that are not physically part of the product.
Do not change anything on the product itself — keep its shape, colors, materials, labels, and details exactly as they are.
```

---

## Seeding the Prompt into the Database

```python
import sqlite3, time, json

PROMPT_CONTENT = """Clean up the image, center the product, and place it on a pure white #FFFFFF background.
Remove all text, graphics, logos, stickers, watermarks, and overlays that are not physically part of the product.
Do not change anything on the product itself — keep its shape, colors, materials, labels, and details exactly as they are."""

DB = "/data/data/com.termux/files/home/open-webui-data/webui.db"
now = int(time.time())

conn = sqlite3.connect(DB)
conn.execute("""
    INSERT OR REPLACE INTO prompt
    (id, command, user_id, name, content, data, meta, is_active, version_id, tags, created_at, updated_at)
    VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
""", (
    "product-clean",
    "product-clean",
    "system",
    "Product Image Cleaner (#FFFFFF White Background)",
    PROMPT_CONTENT,
    "{}",
    json.dumps({"description": "Centers product on pure white background, removes all overlays and text."}),
    1,
    "v1",
    json.dumps([{"name": "e-commerce"}, {"name": "product-editing"}]),
    now,
    now,
))
conn.commit()
conn.close()
print("Prompt seeded.")
```

---

## How to Use It

1. Start a new chat
2. **Disable `get_current_timestamp`** tool (image models don't support tool use — see [TROUBLESHOOTING.md](TROUBLESHOOTING.md#openrouter-tool-use-error))
3. Select an image generation model (e.g. `openai/gpt-4o-image`, `google/gemini-2.0-flash-exp:image-generation`)
4. Upload your product photo (paperclip / attach icon)
5. Type `/product-clean` — the full prompt auto-fills
6. Send

---

## Compatible Models on OpenRouter

| Model | Notes |
|---|---|
| `openai/gpt-4o-image` | Strong instruction following, good at removing overlays |
| `google/gemini-2.0-flash-exp:image-generation` | Fast, free tier available |
| `openai/gpt-5` | If available on your plan |

**Avoid** models labeled as text-only or chat-only — they will return the tool-use error described in [TROUBLESHOOTING.md](TROUBLESHOOTING.md).
