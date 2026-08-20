# ODS GPU Share — Laptop + Desktop Setup

Run your local AI models on a **powerful desktop** (GPU + RAM) and use them from a
**laptop** (or any other machine on your network).

This repo documents a working setup for sharing one machine's compute across
machines using **ODS (Osmantic Deployment System)** — a free, local AI server
stack (LLM inference, chat UI, dashboard, voice, agents, workflows, RAG).

The short version: the desktop runs the model; the laptop just sends requests.

---

## Architecture

```
┌─────────────────────────────┐         ┌──────────────────────────────┐
│  LAPTOP  (client)           │   LAN   │  DESKTOP  (server, GPU)      │
│                             │ ◄─────► │                              │
│  opencode / Open WebUI      │         │  ODS stack                   │
│  sends requests to          │         │    ├─ llama-server (GPU)     │
│  http://<desktop-ip>:11434  │         │    │   port 11434  (model)   │
│                             │         │    ├─ Open WebUI port 3000   │
│                             │         │    ├─ Dashboard  port 3001   │
│                             │         │    └─ 15 services total      │
└─────────────────────────────┘         └──────────────────────────────┘
```

- **Inference happens only on the desktop** (its GPU VRAM + CPU + RAM).
- The laptop is a thin client — it needs almost no resources.
- Bonus: a desktop GPU's VRAM holds **large contexts easily**, so the
  16K-token context ceiling you hit on a laptop goes away.

---

## What you need on each machine

| Machine | Requirement |
|---|---|
| **Desktop** | Docker Desktop with WSL2 (Windows) **or** Docker (Linux), 16 GB+ RAM, an NVIDIA or AMD GPU (recommended) |
| **Laptop** | opencode (or just a browser), network access to the desktop |

---

## Desktop setup (the "other computer" — do this first)

### 1. Install Docker

**Windows:**
- Install [Docker Desktop](https://www.docker.com/products/docker-desktop/) with the **WSL2 backend**.
- Docker Desktop requires **Windows Subsystem for Linux 2**. If missing:

  ```powershell
  winget install --id Microsoft.WSL --accept-source-agreements --accept-package-agreements
  ```

  Then enable the required Windows features (elevated shell):

  ```powershell
  wsl --install --no-distribution
  ```

  **A Windows restart is required** after enabling WSL2/Virtual Machine Platform.

- After reboot, start Docker Desktop and wait for the whale icon (engine ready).

**Linux (alternative):** install Docker Engine + Docker Compose per your distro's docs.

### 2. Install ODS

Open a normal (non-Administrator) PowerShell and run:

```powershell
$ProgressPreference = "SilentlyContinue"
$odsSrc = Join-Path $env:TEMP ("ods-install-" + [guid]::NewGuid().ToString("N"))
$odsZip = Join-Path $odsSrc "ods-main.zip"
New-Item -ItemType Directory -Path $odsSrc | Out-Null
Invoke-WebRequest "https://github.com/Osmantic/ODS/archive/refs/heads/main.zip" -OutFile $odsZip
Expand-Archive -LiteralPath $odsZip -DestinationPath $odsSrc -Force
cd (Get-ChildItem -LiteralPath $odsSrc -Directory | Select-Object -First 1).FullName
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\install.ps1 -Lan
```

The `-Lan` flag binds services to `0.0.0.0` so other machines can reach them.

> Prefer the stable release over `main`? Follow the
> [Windows Quickstart](https://github.com/Osmantic/ODS/blob/main/ods/docs/WINDOWS-QUICKSTART.md)
> or pin a tagged release. Install time is 10–30 minutes (model download).

### 3. Find the desktop's IP address

```powershell
ipconfig
```

Look for the IPv4 of your active adapter, e.g. `192.168.1.50`. **This is
`<desktop-ip>` everywhere below.**

### 4. Verify the model API is reachable

From the **laptop**, open in a browser (or PowerShell):

```
http://<desktop-ip>:11434/v1/models
```

You should see the desktop's model list as JSON (e.g. a 32B Qwen GGUF).

You can also browse the desktop's own web UI from the laptop:

- Chat UI: `http://<desktop-ip>:3000`
- Dashboard: `http://<desktop-ip>:3001`

---

## Laptop setup

### Option A — opencode (recommended)

opencode's `llama-server` provider just needs to point at the desktop instead of
`localhost`. Edit your global opencode config
(`~/.config/opencode/opencode.json`):

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "llama-server": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "llama-server (desktop)",
      "options": {
        "baseURL": "http://<desktop-ip>:11434/v1",
        "apiKey": "no-key"
      },
      "models": {
        "<model-file-name>.gguf": {
          "name": "<friendly-name>",
          "limit": { "context": 128000, "output": 32768 }
        }
      }
    }
  }
}
```

- `<desktop-ip>` is the desktop's LAN IP (step 3 above).
- `<model-file-name>.gguf` is the exact GGUF filename the desktop serves —
  check it with the `http://<desktop-ip>:11434/v1/models` endpoint.
- Set `"model": "llama-server/<model-file-name>.gguf"` if you want it as the
  default, or pick it in the model switcher per session.
- Your existing cloud provider (e.g. deepseek) is unaffected — opencode merges
  configs, so both stay available.

Verify:

```powershell
opencode models llama-server
```

### Option B — browser only

No install needed: open `http://<desktop-ip>:3000` (chat) or `:3001` (dashboard)
from the laptop.

---

## Security notes (important)

- **`-Lan` / `BIND_ADDRESS=0.0.0.0` exposes the model API to your whole LAN.**
  On a trusted home network this is fine. On shared/public networks, use
  **Tailscale** instead:
  - Install Tailscale on both machines, then set the desktop's baseURL to the
    Tailscale IP (e.g. `http://100.x.x.x:11434/v1`) and bind ODS to that address.
  - ODS ships a guide: `ods/docs/TAILSCALE.md`.
- ODS binds to `127.0.0.1` by default *for security* — only use `-Lan` when you
  actually want LAN access.
- A network-bound ODS install keeps auth enabled on its web UIs and prompts for
  an admin account on first use.

---

## Troubleshooting

### WSL2 / Docker

| Problem | Fix |
|---|---|
| `wsl: WSL is finishing an upgrade` or "WSL is not installed" | `winget install --id Microsoft.WSL`, then `wsl --install --no-distribution`, then **reboot** |
| "Virtual Machine Platform" / "virtualization not enabled" | Enable the feature (needs reboot) and confirm virtualization is ON in BIOS |
| Docker engine not responding | Start Docker Desktop, wait for the whale icon, then run `docker info` |
| Installer warns about WSL2 | Re-run `wsl --install` and restart before installing ODS |

### Model / context / RAM (laptop-side lessons)

- **The model + context must fit the machine's RAM.** On a 16 GB laptop, a
  model like phi-4-mini at 128K context needs a ~16 GB KV cache and will not
  load (llama-server returns 503). Lower the context:
  - Edit `C:\Users\mabek\ods\.env`: `CTX_SIZE=16384`, `MAX_CONTEXT=16384`
  - Restart: `.\ods.ps1 restart llama-server`
- **opencode's agent prompt can exceed a small context.** opencode always sends
  your rules + skills + its own system prompt (~19K tokens). On a GPU desktop
  this stops being a problem (big VRAM/context). On a CPU laptop it can exceed
  a 16K context — keep the desktop on the client role.
- **ODS bundles its own opencode binary** at `~/.opencode/bin/opencode.exe`.
  Keep it updated to match your opencode version, or older binaries can reject
  newer config keys (`tool_output`, `references`).
- **opencode merges config files** (`opencode.json` + `opencode.jsonc`) rather
  than replacing them — a project `"instructions": []` won't strip global rules.

### Connectivity

| Symptom | Check |
|---|---|
| `Unable to connect` from laptop to `:11434` | Desktop ODS running? `BIND_ADDRESS=0.0.0.0` set? Same subnet / firewall? |
| Model API reachable but model "not found" | Model file name in opencode config must match the exact GGUF filename the desktop serves |
| `request exceeds available context size` | Lower the model context in `.env`, or use a model that fits the desktop's VRAM |

---

## Useful commands (desktop)

Run from the ODS runtime directory (`%USERPROFILE%\ods` on Windows):

```powershell
.\ods.ps1 status                # health checks + GPU status
.\ods.ps1 model list            # available model tiers
.\ods.ps1 model swap T3         # switch to a bigger tier (e.g. 32B)
.\ods.ps1 logs llama-server     # tail model server logs
.\ods.ps1 restart llama-server  # apply .env changes
```

---

## Files

- `README.md` — this document.
- `docs/` — (add per-machine cheat sheets or diagrams here as needed).

---

*Model server: ODS 2.6.x · Docs: https://github.com/Osmantic/ODS*
