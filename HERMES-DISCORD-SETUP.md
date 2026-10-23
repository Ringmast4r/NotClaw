# Local AI Assistant — Full Setup (inference rig)

> Your private AI ("**Tzu**") that you talk to over **Discord**, running entirely on your own
> 5× RTX 3070 GPUs. No cloud, no API fees, web search included.
> Last updated: 2026-06-11

---

## The big picture: it's a 3-layer stack

You are **not** choosing between these — they **stack**. Each layer needs the one below it.

```
┌─────────────────────────────────────────────────────────────┐
│  YOU  →  Discord (the app you type in, "Tzu" bot)           │
├─────────────────────────────────────────────────────────────┤
│  LAYER 3 — HARNESS:  Hermes Agent                           │  the "connector / body"
│    Connects Discord to the model, gives it web search,      │
│    memory, tools. Without this it's just a raw model.       │
├─────────────────────────────────────────────────────────────┤
│  LAYER 2 — ENGINE:   Ollama                                 │  the "life support"
│    Loads the model onto your 5 GPUs and runs the math.      │
├─────────────────────────────────────────────────────────────┤
│  LAYER 1 — MODEL:    qwen3-coder:30b                        │  the "brain"
│    The neural net that actually thinks / decides to search. │
├─────────────────────────────────────────────────────────────┤
│  HARDWARE:  5× NVIDIA RTX 3070  (40 GB total VRAM)          │
└─────────────────────────────────────────────────────────────┘
```

**One message's journey:**
`You type in Discord → Hermes receives it → sends to Ollama → qwen3-coder thinks,
decides to web_search → Hermes runs the search (Google/Yandex/etc) → qwen writes the
answer → Hermes sends it back to your Discord DM.`

---

## The 3 layers in detail

### LAYER 1 — Model (the brain): `qwen3-coder:30b`
- **What:** Qwen3-Coder-30B-A3B — a Mixture-of-Experts model (30B total, ~3B active per token, so it's *fast*). Built for agentic tool use.
- **Why this one:** fits 100% on your GPUs (~22 GB), has 256K context (Hermes requires ≥64K), and reliably calls tools. Bigger Qwens (235B/480B) need 3–6× your VRAM. `qwen3:32b` was rejected (only 40K context). `hermes3:70b` was too slow (~4 min/reply, spills to CPU).
- **Runs at:** 100% GPU, spread across all 5 cards via `OLLAMA_SCHED_SPREAD`.
- **Swap it:** edit `~/.hermes/config.yaml` → `model.default:` → any model from `ollama list`, then `systemctl --user restart hermes-gateway`.

### LAYER 2 — Engine: `Ollama 0.30.7`
- **Serves models** on `http://localhost:11434`. systemd service (`ollama`, system-level, enabled).
- **Tuning** (`/etc/systemd/system/ollama.service.d/`):
  - `override.conf`: `SCHED_SPREAD=true` (use all 5 GPUs), `KV_CACHE_TYPE=q8_0`, `KEEP_ALIVE=24h` (pin model in VRAM), `CONTEXT_LENGTH=32768`, `NUM_PARALLEL=1`, `MULTIUSER_CACHE=true`.
  - `gpu-wait.conf`: makes Ollama **wait for the GPUs on boot** (fixes the cold-boot "everything lands on CPU" bug).
- **Boot preload:** `ollama-preload.service` warms qwen3-coder into VRAM after every reboot.
- ⚠️ **Cold load is slow (~4–5 min)** on this version — the first load reads 18 GB off disk. Don't interrupt it (interrupting *aborts* the load). Once warm, it stays 24h.

### LAYER 3 — Harness: `Hermes Agent v0.16.0`
- **What:** NousResearch's agent framework (MIT). Lives in `~/.hermes/`. This is the part that gives you Discord, web search, memory, and tools.
- **Points at Ollama:** `~/.hermes/config.yaml` → `provider: ollama`, `base_url: http://localhost:11434/v1`.
- **Runs as:** `hermes-gateway` (systemd **user** service, linger enabled → survives logout/reboot).
- **Web search:** `web.backend: ddgs` — keyless, FREE, multi-engine. Searches **Google, Brave, DuckDuckGo, Yandex, Startpage, Mojeek** and aggregates. (Patched in `~/.hermes/hermes-agent/plugins/web/ddgs/provider.py`.)
- **Skills:** trimmed to a minimal set (the default 86 bloated the prompt and confused the model — kept web/essential only).

---

## How to USE it

- **Talk to it:** open Discord → DM the **Tzu** bot. Type normally (no @mention needed in DMs).
- **From your phone:** same Discord DM works anywhere.
- **It's locked to you only** (`DISCORD_ALLOWED_USERS` = your Discord ID). Nobody else can use it.
- ⚠️ **Keep Tzu in your "Hooked" server** — Discord requires a shared server for the bot to DM you. Don't kick it out or DMs break. You never have to *use* the server; just leave the bot there.
- **First message after a reboot is slow (~30–50s)** while the model warms; after that it's fast.

---

## Managing it (cheat sheet)

```bash
# Status
ollama ps                                  # is the model loaded on GPU?
systemctl is-active ollama                 # engine up?
systemctl --user is-active hermes-gateway  # harness up?
nvidia-smi                                  # GPU usage

# Restart layers
sudo systemctl restart ollama              # engine (warm-restart fixes GPU discovery)
systemctl --user restart hermes-gateway    # harness (after any config change)

# Logs
journalctl --user -u hermes-gateway -f                 # live harness log
tail -f ~/.hermes/logs/agent.log                       # model turns / web searches

# Swap model / config
nano ~/.hermes/config.yaml                 # model.default, web backend, etc.
ollama list                                # available models

# Manually warm the model (after reboot, if needed) — DO NOT interrupt
curl -s -m 720 http://localhost:11434/api/generate \
  -d '{"model":"qwen3-coder:30b","prompt":"hi","keep_alive":"24h"}' >/dev/null
```

---

## Key files

| What | Where |
|---|---|
| Hermes config (model, web, etc.) | `~/.hermes/config.yaml` |
| Secrets (Discord token, allowlist) | `~/.hermes/.env` (perms 600) |
| Multi-engine search patch | `~/.hermes/hermes-agent/plugins/web/ddgs/provider.py` |
| Ollama tuning | `/etc/systemd/system/ollama.service.d/override.conf` |
| Ollama GPU-wait + preload | `…/ollama.service.d/gpu-wait.conf`, `/etc/systemd/system/ollama-preload.service` |
| Model weights | `/usr/share/ollama/.ollama/models/` |

---

## History / decisions (so you don't re-litigate them)

- **Discord, not Signal.** Signal was tried first (via `signal-cli`) but it **crashes** (`getServerGuid` NullPointerException) on Note-to-Self messages synced from the Windows Signal Desktop. That's a `signal-cli` bug, not fixable by us. Discord uses an official API and "just works."
- **qwen3-coder, not hermes3.** The `hermes3` *model* is conversational; `qwen3-coder` is agentic and reliably calls tools. (Note: "Hermes" is confusingly both the framework AND a model family — we use the framework, NOT the model.)
- **Ollama, not optional.** Ollama is the engine; it's not an alternative to Hermes — Hermes runs *on top of* it.
- **Web search is keyless (ddgs).** Brave's API is no longer free ($5/1k). ddgs covers Google/Yandex/etc for free.

---

## Other models you have (all local, via `ollama list`)
`qwen3-coder:30b` (current) · `qwen3:32b` · `qwen2.5-coder:32b/14b` · `hermes3:8b/70b` ·
`gemma3:27b` · `gemma3-tools:27b` (these two can also see images).
