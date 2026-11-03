# ai-local

> **Moving to a new Mac?** [GETTING-STARTED.md](GETTING-STARTED.md) describes the exact current setup
> (Ollama 0.40.2, Open WebUI 0.12.5 + audioop-lts, the three launch agents in [`launchd/`](launchd/),
> backup/restore and troubleshooting).

Run local LLMs on a Mac and reach them from any device on your network at **http://ai.local**.

- **[Ollama](https://ollama.com)** runs the models (stays on `127.0.0.1:11434`).
- **[Open WebUI](https://github.com/open-webui/open-webui)** is the chat UI, served on port 80 with its own login.
- **Bonjour/mDNS** publishes `ai.local` → the Mac's current LAN IP, and follows it if the IP changes.

Everything is managed by one script, `aictl`. No Docker.

## Getting started

Requirements: macOS on Apple Silicon, [Homebrew](https://brew.sh), and enough free disk for the
models you pick (see `models.txt`).

```sh
brew install uv
git clone https://github.com/davmoz/ai-local.git
cd ai-local
./aictl install
```

`install` will:

1. download Ollama.app into `/Applications` if it isn't there (signature-checked) and add it as a login item,
2. install Open WebUI with `uv tool` (pinned version, Python 3.12),
3. link `aictl` and `ollama` into `~/.local/bin`,
4. install two launch agents (`local.ai.webui`, `local.ai.mdns`) that start at login and restart on failure,
5. pull every model listed in `models.txt`.

Then open **http://ai.local** (or http://localhost). The first account you create becomes the
admin; turn off sign-ups for others under *Admin Panel → Settings → General*.

## Day to day

```sh
aictl status          # services, ai.local resolution, installed models
aictl update          # newer Ollama.app + model updates; tells you if Open WebUI has a new release
aictl models --prune  # make installed models match models.txt
aictl restart
aictl logs            # ~/Library/Logs/ai-local/
aictl uninstall       # removes agents + Open WebUI, keeps models and chat history
```

## Choosing models

Edit `models.txt` (one Ollama tag per line) and run `aictl models --prune`.
Browse <https://ollama.com/search>. Rule of thumb: the model file should be well under your
unified memory — on a 36 GB Mac, stay at or below ~24 GB and load one big model at a time.
Prefer `-mlx` tags on Apple Silicon.

## Where things live

| What | Path |
|---|---|
| Model weights | `~/.ollama` (never in this repo) |
| Chat history, users, settings | `~/Library/Application Support/ai-local/webui` |
| Session secret | `~/Library/Application Support/ai-local/webui-secret` |
| Launch agents | `~/Library/LaunchAgents/local.ai.*.plist` |
| Logs | `~/Library/Logs/ai-local/` |

## Notes

- Only Open WebUI (behind login) is exposed to the LAN; the Ollama API stays on localhost.
- `ai.local` resolves out of the box on macOS/iOS. Windows needs Bonjour; Linux needs Avahi + nss-mdns.
- Port 80 works without root on macOS. If it is taken: `AI_LOCAL_PORT=8080 ./aictl install` (the URL becomes `http://ai.local:8080`).
