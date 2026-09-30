# Installation Guide — Open WebUI on Termux

Complete step-by-step installation of Open WebUI v0.11.4 on Android/Termux with OpenRouter as the AI backend.

---

## Phase 1 — Termux Setup

```bash
# Update packages
pkg update && pkg upgrade -y

# Install required system packages
pkg install -y python python-pip git curl wget libffi openssl libjpeg-turbo

# Install Termux:API (for wake lock — keep server alive)
# Also install the Termux:API Android app from F-Droid
pkg install -y termux-api

# Create bin directory
mkdir -p ~/bin
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

---

## Phase 2 — Python Virtual Environment

Open WebUI requires Python 3.11.

```bash
# Check Python version
python --version   # Should be 3.11.x

# Create venv
python -m venv ~/open-webui-env

# Activate
source ~/open-webui-env/bin/activate
```

---

## Phase 3 — Install Open WebUI

```bash
# Pre-install build tools
pip install --upgrade pip setuptools wheel

# Install open-webui
# Note: Do NOT install numpy/scipy — they require glibc unavailable on Android Bionic
pip install open-webui==0.11.4 \
    --no-deps 2>/dev/null || true

# Install dependencies manually (order matters on Android)
pip install \
    bcrypt \
    "cryptography==50.0.1" \
    "starsessions==2.2.1" \
    redis hiredis \
    "pycrdt==0.14.8" \
    "tiktoken==0.14.0" \
    regex \
    langchain-core langchain langchain-text-splitters \
    ldap3 \
    "mcp==1.27.2" \
    black \
    Pillow \
    beautifulsoup4 \
    markdown \
    langdetect \
    ftfy pytz python-dateutil \
    azure-identity \
    pyzipper \
    typer

# Install open-webui itself last
pip install open-webui==0.11.4
```

---

## Phase 4 — Android/Bionic Compatibility Fixes

Android uses the Bionic C library, not glibc. Several packages need workarounds.

### 4.1 LD_PRELOAD Fix

Every Python process must preload libpython to resolve C extension symbols:

```bash
export LD_PRELOAD=/data/data/com.termux/files/usr/lib/libpython3.11.so
```

This is set automatically in all `owui-*` scripts.

### 4.2 numpy Stub

Real numpy requires `libm.so.6` and OpenBLAS which don't exist on Android Bionic. Create a stub:

```bash
cat > ~/open-webui-env/lib/python3.11/site-packages/numpy/__init__.py << 'EOF'
"""numpy stub — real numpy not available on Android Bionic (no libm.so.6).
Open WebUI slim mode does not need numpy for image-generation-only use."""
__version__ = "1.26.4"

class _StubArray:
    pass

ndarray = _StubArray

def array(*a, **kw): return _StubArray()
def zeros(*a, **kw): return _StubArray()
def ones(*a, **kw): return _StubArray()
def float32(*a, **kw): return 0.0
def int32(*a, **kw): return 0

float32 = float
int32 = int
int64 = int
uint8 = int
bool_ = bool
EOF
```

### 4.3 chromadb Stub

chromadb requires SQLite extensions not available on Android. Since we use `USE_SLIM_DOCKER=true`, we just need a stub so the import doesn't crash:

```bash
mkdir -p ~/open-webui-env/lib/python3.11/site-packages/chromadb
cat > ~/open-webui-env/lib/python3.11/site-packages/chromadb/__init__.py << 'EOF'
"""chromadb stub — not needed in slim mode (USE_SLIM_DOCKER=true)."""
class Client:
    pass
class Settings:
    def __init__(self, **kw): pass
def HttpClient(**kw): return Client()
def EphemeralClient(**kw): return Client()
EOF
```

### 4.4 re2 Stub

```bash
cat > ~/open-webui-env/lib/python3.11/site-packages/re2.py << 'EOF'
"""re2 stub — wraps stdlib re. google-re2 C extension not available on Android."""
from re import *
from re import compile, search, match, fullmatch, findall, finditer, sub, subn, split, escape, purge, error
EOF
```

---

## Phase 5 — Data Directory and Configuration

```bash
mkdir -p ~/open-webui-data

# Generate a secret key
SECRET=$(python -c "import secrets; print(secrets.token_hex(32))")

cat > ~/open-webui-data/.env << EOF
# Open WebUI Environment Configuration
# DO NOT commit this file — contains secrets

WEBUI_NAME="Image Studio"
DATA_DIR=$HOME/open-webui-data
FRONTEND_BUILD_DIR=$HOME/open-webui-env/lib/python3.11/site-packages/open_webui/frontend

WEBUI_SECRET_KEY=$SECRET

HOST=127.0.0.1
PORT=8080

# Slim mode — remote inference only, no local models
USE_SLIM_DOCKER=true
ENABLE_OLLAMA_API=false
ENABLE_OPENAI_API=false

# Disable heavy optional features
ENABLE_RAG_WEB_SEARCH=false
ENABLE_SEARCH_QUERY=false
ENABLE_COMMUNITY_SHARING=false
ENABLE_MESSAGE_RATING=false
ENABLE_LOGIN_FORM=true
ENABLE_SIGNUP=true

GLOBAL_LOG_LEVEL=INFO
EOF
```

---

## Phase 6 — Run Migrations

```bash
export LD_PRELOAD=/data/data/com.termux/files/usr/lib/libpython3.11.so
export USE_SLIM_DOCKER=true
source ~/open-webui-env/bin/activate

# Run migrations (creates webui.db with all tables)
cd ~/open-webui-data
python -c "
import os
os.environ['DATA_DIR'] = os.path.expanduser('~/open-webui-data')
os.environ['USE_SLIM_DOCKER'] = 'true'
from open_webui.internal.db import Base, engine
Base.metadata.create_all(bind=engine)
print('Migrations done.')
"
```

If the above fails, just start the server once — it runs migrations automatically at startup.

---

## Phase 7 — Install Scripts

Copy all scripts from `bin/` to `~/bin/` and make them executable:

```bash
cp bin/owui-* ~/bin/
chmod +x ~/bin/owui-*
```

---

## Phase 8 — Seed OpenRouter Pipe and Prompt Preset

```bash
# The pipe code lives at ~/open-webui-data/open_webui_openrouter_pipe_bundled.py
# Download it:
curl -Lo ~/open-webui-data/open_webui_openrouter_pipe_bundled.py \
  https://raw.githubusercontent.com/rbb-dev/Open-WebUI-OpenRouter-pipe/main/openrouter_pipe.py

# Then seed it via the startup API or directly into the DB:
# (The owui-start script handles this automatically on first run)
```

See [`docs/PIPE_SETUP.md`](PIPE_SETUP.md) for full pipe configuration including API key setup.

---

## Phase 9 — Start the Server

```bash
# In a Termux terminal — KEEP THIS WINDOW OPEN
owui-watchdog
```

Wait for: `INFO: Application startup complete.`

Then open **http://127.0.0.1:8080** in your browser.

**First user to register becomes admin.**

---

## Phase 10 — Configure OpenRouter API Key

1. Go to **Admin Panel → Functions**
2. Find "OpenRouter Pipe" — toggle ON
3. Click the **⚙️ gear icon** on the pipe
4. Paste your OpenRouter API key
5. Click Save

The key is stored encrypted in `webui.db` using your `WEBUI_SECRET_KEY`. It never touches any script or config file.
