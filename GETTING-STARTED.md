# Getting started — reproduce ai.local on a new Mac

This is the exact setup running on the Mac Studio as of 2026-11-03. Follow it top to bottom on a
new Apple Silicon Mac and you end up with the same thing: **http://ai.local** on the LAN, Open WebUI
in front of Ollama, two Qwen models, one model kept loaded in memory at a time.

| Piece | Version / value |
|---|---|
| Ollama.app | 0.40.2 (`/Applications/Ollama.app`, login item) |
| Models | `qwen3.6:35b-a3b` (nvfp4, MLX runner, 23 GB) and `qwen3.8:27b-mlx` (nvfp4, 18 GB) |
| Open WebUI | 0.12.5 via `uv tool`, Python 3.13, plus `audioop-lts` |
| Web port | 80 on `0.0.0.0` (no root needed on macOS) |
| Hostname | `ai.local` published over Bonjour/mDNS |
| Ollama API | `127.0.0.1:11434` only (not exposed to the LAN) |
| Keep-alive | `OLLAMA_KEEP_ALIVE=-1`, `OLLAMA_MAX_LOADED_MODELS=1` |
| Data | `~/Library/Application Support/ai-local/webui` (`webui.db`, `vector_db/`, `uploads/`) |
| Session secret | `~/Library/Application Support/ai-local/webui-secret` (generated on first start, never commit) |
| Logs | `~/Library/Logs/ai-local/ai-local-webui.log`, `ai-local-mdns.log` |

The paths below assume the user is `damo` and the repo is cloned to `~/github/ai-local`
(the launch agents point at `bin/` in that checkout). If either differs, edit the paths in
`launchd/*.plist` before copying them.

## 1. Prerequisites

```sh
# Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
eval "$(/opt/homebrew/bin/brew shellenv)"

# uv (installs/runs Open WebUI)
brew install uv

# this repo
mkdir -p ~/github && git clone git@github.com:davmoz/ai-local.git ~/github/ai-local
```

Ollama.app: download from <https://ollama.com/download> (or let `./aictl install` fetch it), move
it to `/Applications`, open it once, and keep it as a login item
(*System Settings → General → Login Items*). Link the CLI:

```sh
mkdir -p ~/.local/bin
ln -sf /Applications/Ollama.app/Contents/Resources/ollama ~/.local/bin/ollama
ln -sf ~/github/ai-local/aictl ~/.local/bin/aictl
export PATH="$HOME/.local/bin:/opt/homebrew/bin:$PATH"   # add to ~/.zshrc
```

## 2. Keep-alive / one model at a time

Ollama.app reads its settings from the launchd environment. `local.ai.ollama-env` sets them at
every login:

```sh
cp ~/github/ai-local/launchd/local.ai.ollama-env.plist ~/Library/LaunchAgents/
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/local.ai.ollama-env.plist
launchctl getenv OLLAMA_KEEP_ALIVE          # -1
launchctl getenv OLLAMA_MAX_LOADED_MODELS   # 1
```

Then **quit and reopen Ollama.app** so it picks the variables up (it only reads them at start).
`-1` = a loaded model never unloads; `1` = loading the other model evicts the first.

## 3. Pull the models

```sh
ollama pull qwen3.6:35b-a3b
ollama pull qwen3.8:27b-mlx
ollama list
```

## 4. Install Open WebUI (pinned)

```sh
uv tool install open-webui@0.12.5 --force --with audioop-lts
```

`audioop-lts` is required: uv picks Python 3.13, which no longer ships `audioop`, and Open WebUI
then crashes at start with `ModuleNotFoundError: No module named 'pyaudioop'`.

## 5. Launch agents (web UI on port 80 + mDNS)

```sh
mkdir -p ~/Library/Logs/ai-local
cp ~/github/ai-local/launchd/local.ai.webui.plist ~/github/ai-local/launchd/local.ai.mdns.plist ~/Library/LaunchAgents/
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/local.ai.webui.plist
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/local.ai.mdns.plist
```

- `local.ai.webui` runs `bin/ai-local-webui`, which exports `DATA_DIR="$HOME/Library/Application Support/ai-local/webui"`,
  `OLLAMA_BASE_URL=http://127.0.0.1:11434`, `WEBUI_NAME=AI`, `WEBUI_URL=http://ai.local`,
  `WEBUI_SECRET_KEY` (read from the `webui-secret` file), telemetry off, and runs
  `open-webui serve --host 0.0.0.0 --port 80`. First start takes about a minute.
- `local.ai.mdns` runs `bin/ai-local-mdns`, which publishes `ai.local` → the Mac's current LAN IP
  with `dns-sd -P` and re-registers when the IP changes.

Exact contents of the three agents as installed (also in [`launchd/`](launchd/); no secrets live in
them — the session secret is in the `webui-secret` file):

`~/Library/LaunchAgents/local.ai.webui.plist`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>local.ai.webui</string>
  <key>ProgramArguments</key><array><string>/Users/damo/github/ai-local/bin/ai-local-webui</string></array>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>ThrottleInterval</key><integer>10</integer>
  <key>StandardOutPath</key><string>/Users/damo/Library/Logs/ai-local/ai-local-webui.log</string>
  <key>StandardErrorPath</key><string>/Users/damo/Library/Logs/ai-local/ai-local-webui.log</string>
  <key>EnvironmentVariables</key>
  <dict><key>PATH</key><string>/Users/damo/.local/bin:/opt/homebrew/bin:/usr/bin:/bin:/usr/sbin:/sbin</string></dict>
</dict>
</plist>
```

`~/Library/LaunchAgents/local.ai.mdns.plist`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>local.ai.mdns</string>
  <key>ProgramArguments</key><array><string>/Users/damo/github/ai-local/bin/ai-local-mdns</string></array>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>ThrottleInterval</key><integer>10</integer>
  <key>StandardOutPath</key><string>/Users/damo/Library/Logs/ai-local/ai-local-mdns.log</string>
  <key>StandardErrorPath</key><string>/Users/damo/Library/Logs/ai-local/ai-local-mdns.log</string>
  <key>EnvironmentVariables</key>
  <dict><key>PATH</key><string>/Users/damo/.local/bin:/opt/homebrew/bin:/usr/bin:/bin:/usr/sbin:/sbin</string></dict>
</dict>
</plist>
```

`~/Library/LaunchAgents/local.ai.ollama-env.plist`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
<key>Label</key><string>local.ai.ollama-env</string>
<key>ProgramArguments</key><array><string>/bin/sh</string><string>-c</string>
<string>launchctl setenv OLLAMA_KEEP_ALIVE -1; launchctl setenv OLLAMA_MAX_LOADED_MODELS 1</string></array>
<key>RunAtLoad</key><true/>
</dict></plist>
```

## 6. First login

Open <http://ai.local> (or <http://localhost>). The first account becomes admin. When everyone who
needs an account has one: *Admin Panel → Settings → General* → turn **Enable New Sign Ups** off and
set **Default User Role** to `pending` (or `user`). Leaving sign-up on with default role `admin`
lets anyone on the LAN create an admin account.

## 7. Verify

```sh
curl -s http://ai.local/api/config | python3 -m json.tool | head   # name "AI", version 0.12.5
curl -s http://127.0.0.1:11434/api/version                           # Ollama version
ollama ps            # loaded model, PROCESSOR 100% GPU, RUNNER mlx, UNTIL Forever
dns-sd -G v4 ai.local                                                 # resolves to the Mac's LAN IP (Ctrl-C)
aictl status
```

Expected performance on the Mac Studio (measured 2026-11-03, via Ollama):

| Model | Speed |
|---|---|
| `qwen3.6:35b-a3b` | ~130 tok/s prose, 144–149 tok/s code |
| `qwen3.8:27b-mlx` | ~42–50 tok/s |
| Warm first token | < 0.2 s |
| Switching model | ~7 s (only one stays loaded) |

## 8. Backup and restore

Everything Open WebUI knows (users, chats, settings, knowledge) is in
`~/Library/Application Support/ai-local/webui`. Model weights live in `~/.ollama` and are re-pulled
instead of backed up.

```sh
D="$HOME/Library/Application Support/ai-local"
# backup (consistent copy while running)
sqlite3 "$D/webui/webui.db" ".backup '$HOME/webui-$(date +%F).db'"
# full backup incl. vector_db/uploads + session secret
launchctl bootout gui/$(id -u)/local.ai.webui
tar -czf ~/ai-local-data-$(date +%F).tgz -C "$HOME/Library/Application Support" ai-local
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/local.ai.webui.plist
```

Restore on the new Mac (after steps 1–5):

```sh
launchctl bootout gui/$(id -u)/local.ai.webui
tar -xzf ~/ai-local-data-YYYY-MM-DD.tgz -C "$HOME/Library/Application Support"
#   or only the DB:  cp ~/webui-YYYY-MM-DD.db "$D/webui/webui.db" && rm -f "$D/webui/webui.db-wal" "$D/webui/webui.db-shm"
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/local.ai.webui.plist
```

Restore with the **same or newer** Open WebUI version; it migrates the DB forward on start, not back.
Take a copy before upgrading (`webui.db.bak-pre-<version>`).

## 9. Troubleshooting

**Open WebUI keeps restarting, log shows `ModuleNotFoundError: ... pyaudioop`** — the `audioop-lts`
extra is missing (Python 3.13). Reinstall: `uv tool install open-webui@0.12.5 --force --with audioop-lts`,
then `launchctl kickstart -k gui/$(id -u)/local.ai.webui`.

**uv says the tool is pinned / installs the wrong version** — the uv receipt pins `open-webui==<version>`,
so `uv tool upgrade` will not move it. Reinstall with the explicit version and **repeat
`--with audioop-lts`** (a reinstall without it drops the extra):
`uv tool install open-webui@<version> --force --with audioop-lts`. Check with
`cat ~/.local/share/uv/tools/open-webui/uv-receipt.toml`.

**Forgot a password** — set a new bcrypt hash in the `auth` table:

```sh
launchctl bootout gui/$(id -u)/local.ai.webui
~/.local/share/uv/tools/open-webui/bin/python - <<'PY'
import bcrypt, getpass, os, sqlite3
db = os.path.expanduser("~/Library/Application Support/ai-local/webui/webui.db")
email = input("email: ")
pw = getpass.getpass("new password: ")
h = bcrypt.hashpw(pw.encode(), bcrypt.gensalt()).decode()
con = sqlite3.connect(db)
n = con.execute("UPDATE auth SET password=? WHERE email=?", (h, email)).rowcount
con.commit(); print("rows updated:", n)
PY
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/local.ai.webui.plist
```

**Sign-up on/off** — *Admin Panel → Settings → General → Enable New Sign Ups* and *Default User Role*.
The saved setting in `webui.db` wins over env vars on restart.

**`ai.local` does not resolve** — `launchctl print gui/$(id -u)/local.ai.mdns | grep state`, check
`~/Library/Logs/ai-local/ai-local-mdns.log`. Windows clients need Bonjour, Linux needs Avahi + nss-mdns.

**Port 80 busy** — `lsof -nP -iTCP:80 -sTCP:LISTEN`. Or run on another port with `AI_LOCAL_PORT=8080`
(add it to the plist `EnvironmentVariables`; URL becomes `http://ai.local:8080`).

**Model unloads / two models in memory** — `launchctl getenv OLLAMA_KEEP_ALIVE` must print `-1`;
if not, `launchctl kickstart gui/$(id -u)/local.ai.ollama-env`, then quit and reopen Ollama.app.
