# Troubleshooting Guide — Open WebUI on Termux

Every error encountered during the installation, with root cause and fix.

---

## Android/Bionic Issues

### `cannot locate symbol "PyExc_ValueError"` (or any `PyExc_*`)

**Context:** Happens when importing `cryptography`, `cffi`, or any C-extension package.

**Root cause:** Android Bionic's dynamic linker doesn't resolve inter-library symbols the same way glibc does. The C extension `.so` files can't find Python's exported symbols at dlopen time.

**Fix:** Set `LD_PRELOAD` before every Python invocation:
```bash
export LD_PRELOAD=/data/data/com.termux/files/usr/lib/libpython3.11.so
```
This forces the linker to load libpython first, making all `PyExc_*` and `Py_*` symbols available globally.

---

### `ImportError: libm.so.6: cannot open shared object file`

**Context:** Happens when importing numpy, scipy, or any package that links against glibc's `libm`.

**Root cause:** manylinux wheels are compiled against glibc. Android Bionic has `libm.so` (not `libm.so.6`). The `.so` files inside numpy's wheel are simply not compatible.

**Fix:** Create a [numpy stub](../INSTALL.md#42-numpy-stub) that satisfies imports without any C code. Open WebUI slim mode doesn't actually execute numpy operations — it just imports it at module load time.

---

### `ImportError: libopenblas64_p-r0-...so: cannot open shared object file`

**Context:** Same as above — numpy/scipy trying to load OpenBLAS.

**Fix:** Same numpy stub fix.

---

### chromadb import fails / `sqlite3` extension errors

**Context:** `open_webui.config` imports chromadb unconditionally (line ~509).

**Root cause:** chromadb uses SQLite extensions that Android's SQLite doesn't support. Also, the manylinux chromadb wheel has the same Bionic incompatibility.

**Fix 1 (correct):** Set `USE_SLIM_DOCKER=true` — this makes Open WebUI skip the chromadb import path entirely.

**Fix 2 (belt-and-suspenders):** Create a [chromadb stub](../INSTALL.md#43-chromadb-stub). Even with slim mode, sometimes the import path is still triggered during module loading.

---

### `ModuleNotFoundError: No module named 're2'`

**Context:** Open WebUI imports `re2` for regex operations.

**Root cause:** The `google-re2` C extension requires glibc — not available on Bionic.

**Fix:** Create a [re2 stub](../INSTALL.md#44-re2-stub) that wraps Python's stdlib `re` module. Functionally identical for Open WebUI's use case.

---

## Open WebUI Startup Issues

### `open_webui has no __main__.py`

**Context:** Trying to run `python -m open_webui`.

**Fix:** Use the binary instead:
```bash
~/open-webui-env/bin/open-webui serve --host 127.0.0.1 --port 8080
```

---

### Pipe installs numpy at startup via `imageio`

**Context:** The OpenRouter pipe's frontmatter had:
```
requirements: aiohttp, cryptography, ..., imageio, numpy, ...
```
At startup, Open WebUI reads this and runs `pip install` for each requirement. `imageio` pulls in `numpy`, which fails on Android.

**Fix:** Edit the `function` table in `webui.db` — set the pipe's `meta` JSON so `requirements:` is an empty string:
```python
import sqlite3, json
conn = sqlite3.connect('webui.db')
# Read current meta, clear requirements in content frontmatter
# Set requirements: (empty) so plugin.py skips pip
```
See [`docs/PIPE_SETUP.md`](PIPE_SETUP.md) for the exact SQL.

---

### Server starts then immediately dies (Android OOM killer)

**Context:** `owui-start` launches the server via `nohup`. The health check passes (`✓ Open WebUI is ready!`) but within 30–60 seconds the process is gone.

**Root cause:** Android's OOM (Out of Memory) killer aggressively kills background processes, especially `nohup`'d ones with no controlling terminal and no wake lock.

**Fix:** Use `owui-watchdog` instead of `owui-start`. The watchdog:
1. Acquires a `termux-wake-lock` (tells Android this is an active process)
2. Runs the server in the **foreground** of a Termux terminal session
3. Auto-restarts if the process is killed

```bash
# In Termux — keep this window open
owui-watchdog
```

Android virtually never kills foreground terminal sessions that hold a wake lock.

---

### `owui-status` reports STOPPED when server is actually running

**Context:** The status script reads a PID file written by `owui-start`. That PID is the `nohup bash` wrapper process, not the actual `open-webui` server. The wrapper exits after launching the server, making the PID stale.

**Fix:** Use `pgrep -f "open-webui serve"` instead of reading the PID file. All three scripts (`owui-start`, `owui-stop`, `owui-status`) were updated to use `pgrep`.

---

### `starsessions` version conflict

**Context:** Open WebUI requires `starsessions>=2.1.3`. Default pip installs 1.x which has a different API.

**Fix:**
```bash
pip install "starsessions==2.2.1"
```

---

### `mcp` version conflict

**Context:** `mcp` (Model Context Protocol) package has breaking changes between minor versions.

**Fix:**
```bash
pip install "mcp==1.27.2"
```

---

## OpenRouter / UI Issues

### `No endpoints found that support tool use. Try disabling "get_current_timestamp"`

**Context:** Happens when chatting with image generation models (GPT-4o Image, Gemini Image, etc.).

**Root cause:** Open WebUI automatically enables a built-in tool called `get_current_timestamp`. When any tool is active, OpenRouter adds `tools: [...]` to the API request. Image generation model endpoints do **not** support the `tools` parameter — they only accept text/image inputs.

**Fix (per-chat):**
1. In the chat input area, click the **Tools** icon (wrench/spanner)
2. Toggle OFF `get_current_timestamp`
3. Now send your image generation request

**Fix (global — admin):**
1. Admin Panel → Tools
2. Find `get_current_timestamp`
3. Toggle the **global default** to OFF

Image models never need a timestamp tool anyway.

---

### OpenRouter pipe not visible in Functions list

**Context:** The pipe was seeded directly into `webui.db` but doesn't appear in the UI.

**Possible causes:**
- The `is_active` or `is_global` field is 0 in the DB
- The pipe code has a syntax error preventing load

**Check:**
```bash
# In Termux
python3 ~/open-webui-data/open_webui_openrouter_pipe_bundled.py
# Should produce no output (clean import)
```

**Fix:**
```python
import sqlite3
conn = sqlite3.connect('/data/data/com.termux/files/home/open-webui-data/webui.db')
conn.execute("UPDATE function SET is_active=1, is_global=1 WHERE id='openrouter_pipe'")
conn.commit()
```
Then restart the server.

---

### `/product-clean` prompt not appearing in chat

**Context:** The prompt preset was seeded in `webui.db` but `/` shortcut in chat doesn't show it.

**Check via UI:** Workspace → Prompts — if it appears there but not in chat shortcut, try refreshing.

**Check via DB:**
```python
import sqlite3
conn = sqlite3.connect('/data/data/com.termux/files/home/open-webui-data/webui.db')
print(conn.execute("SELECT id, command, title FROM prompt").fetchall())
```

---

## Performance Notes

- **Startup time:** ~27 seconds from `owui-watchdog` to HTTP 200 on `/health`. Normal.
- **Memory usage:** ~350–450 MB RSS after startup. Leaves ~3.5 GB for the OS and browser.
- **Inference:** All inference is remote (OpenRouter). Phone CPU/RAM not used for AI computation.
- **Database:** SQLite at `~/open-webui-data/webui.db`. No PostgreSQL needed.
