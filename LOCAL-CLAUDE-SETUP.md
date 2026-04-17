# Local Claude Code — Free, Offline, No API Tokens

A working walkthrough for running a Claude Code-style coding agent **100% locally** on your own hardware. No Anthropic API key, no per-token billing, no data leaving your machine.

---

## ⚠ READ FIRST — single-GPU safety

**If you have one GPU and it also drives your display, the wrong model will black-screen your PC.** No warning, no recovery except a hard reboot. We crashed a friend's 3070 Ti (8 GB, display-driving) on 2026-04-17 by copy-pasting the multi-GPU defaults from an earlier version of this guide. This section exists because of that incident.

The rules, in order of importance:

1. **Set `OLLAMA_GPU_OVERHEAD=2147483648` (2 GiB) in your Ollama systemd service.** This is the single biggest safety improvement available. It reserves 2 GiB of VRAM for your desktop compositor — so even if Ollama tries to over-commit, the display stays alive. Added in Ollama v0.20+. If you learn nothing else from this guide, set this variable.
2. **Pick your model from the VRAM table below BEFORE step 2.** Never pull `qwen2.5-coder:32b` on a card with less than 24 GB dedicated VRAM.
3. **Use `OLLAMA_NUM_PARALLEL=1` on single consumer GPUs.** Each parallel slot reserves its own KV-cache block — `4` is a multi-GPU setting and will blow past VRAM even with a "right-sized" model. This specifically is what crashed the 3070 Ti running a 14b.
4. **Set `OLLAMA_MAX_LOADED_MODELS=1`** so a stray `ollama run <other-model>` can't silently co-load a second model onto your one card.
5. **Use `OLLAMA_KV_CACHE_TYPE=q8_0` (quantized KV cache)** on tight-VRAM cards. Cuts KV-cache memory 2–4x at a small quality cost. Makes the 14b comfortable on 12 GB cards.
6. **The `ccode` launcher below takes the model as an argument** (`CCODE_MODEL=qwen2.5-coder:7b ccode`) so you can't accidentally run the 32b.

If your display has already gone black: hold the power button, reboot, then run the "emergency disable" block at the bottom of this guide BEFORE launching anything.

---

## What this is

Two things came out of the March 31, 2026 Claude Code source map leak and the community rewrites that followed it:

### 1. [ultraworkers/claw-code](https://github.com/ultraworkers/claw-code) — the harness

> "The fastest repo in history to surpass 100K stars."
> Rust implementation of Claude Code's agent harness. Native tool calling, native OpenAI-compat endpoint, no proxy needed.

**185,000+ stars.** Written in Rust from scratch. This is what we actually run.

### 2. [instructkr/claude-code](https://github.com/instructkr/claude-code) — the sibling project

> "An independent Python feature port of alike to Claude Code, entirely rewritten from scratch using oh-my-codex. Educational Purpose only."

**185,000+ stars.** Same fork network as claw-code — they're part of the same community effort. Python port if you prefer Python over Rust.

Both exist because the original Claude Code source map leaked in the NPM package, and a swarm of developers rebuilt a usable open version within days. These two are the most starred and most functional of the result.

---

## What we have running

```
┌─────────────────────────────────────────────────────────────┐
│                     YOUR TERMINAL                           │
│                           │                                 │
│                    type: ccode                              │
│                           ▼                                 │
│    claw-code (Rust)     ←←← the harness                    │
│    gives the LLM tools: read_file, write_file, bash, grep   │
│                           │                                 │
│                           ▼  http://127.0.0.1:11434        │
│    Ollama server        ←←← the model server               │
│    hosts models on your GPUs                                │
│                           │                                 │
│                           ▼                                 │
│    qwen2.5-coder:<size>  ←←← the brain (the model)         │
│    :7b (4.5 GB) / :14b (9 GB) / :32b (19 GB)                │
│    native tool support, no "thinking" overhead              │
│                           │                                 │
│                           ▼                                 │
│    Your GPU(s)          ←←← the hardware                   │
│    single consumer card OR multi-GPU rig                    │
└─────────────────────────────────────────────────────────────┘
```

**No network calls leave the box.** Anthropic doesn't see any of it. You can unplug the internet cable and this still works.

---

## Hardware requirements

**Pick the model that fits your VRAM and STAY in your tier.** "My GPU is close enough" is how you black-screen your PC.

| Your GPU VRAM | Use this model | Pull size | `NUM_PARALLEL` | `KEEP_ALIVE` | Extra env to consider |
|---|---|---|---|---|---|
| 4–6 GB (1650 Super, laptop dGPU, integrated) | `qwen2.5-coder:3b` | ~1.9 GB | `1` | `5m` | `KV_CACHE_TYPE=q8_0`, `GPU_OVERHEAD=1073741824` |
| 8 GB (3060 Ti, 3070/Ti, 4060) | `qwen2.5-coder:7b` | ~4.5 GB | `1` | `5m` | `GPU_OVERHEAD=2147483648` (display GPU) |
| 12 GB (3060 12GB, 4070) | `qwen2.5-coder:7b` or `14b` (tight) | 4.5 / 9 GB | `1` | `5m` | `KV_CACHE_TYPE=q8_0` enables 14b here |
| 16 GB (4060 Ti 16GB, 4070 Ti Super) | `qwen2.5-coder:14b` | ~9 GB | `2` | `30m` | Comfortable; q8_0 KV if also gaming |
| 24 GB (3090, 4090) | `qwen2.5-coder:32b` | 19 GB | `2` | `1h` | First tier that safely fits the 32b |
| 32 GB+ (multi-GPU or A6000) | `qwen2.5-coder:32b` or `llama3.3:70b` | 19 / 40 GB | `2–4` | `24h` | `SCHED_SPREAD=true`, dedicated rig |

> **Smoke-test only** (too small for agent work): `qwen2.5-coder:1.5b` (986 MB), `qwen2.5-coder:0.5b` (398 MB). Good for verifying the stack works on a potato before committing disk/VRAM to a real model.

**Why these numbers matter:**
- Ollama's actual VRAM footprint = model size + KV cache per parallel slot + framebuffer overhead. On a 12 GB card running the 14b with `NUM_PARALLEL=4`, you'll OOM and the driver can hang the display.
- `KEEP_ALIVE` controls how long the model stays resident *after your last request*. `24h` is fine on a dedicated inference box. On a machine you also game or work on, use `5m`–`30m` so VRAM frees up when you alt-tab.

**If your GPU also drives your monitor,** reserve ~1.5–2 GB for the desktop + browser before doing the math. A "12 GB" card realistically offers ~10 GB to Ollama under load.

Any NVIDIA card with CUDA support works. AMD cards work via ROCm but setup is harder. **CPU-only inference** runs but will hammer system RAM — avoid it unless you have 64+ GB of RAM and patience.

---

## Full setup (Linux — tested on Kali, works on Ubuntu/Debian)

### Step 1: Install Ollama (the model server)

```bash
curl -fsSL https://ollama.ai/install.sh | sh
```

### Step 2: Pull the model (CHECK THE VRAM TABLE FIRST)

Do not pull a model larger than your tier. If you are unsure, pick the smaller one.

```bash
# 4–6 GB VRAM (integrated/laptop):
# ollama pull qwen2.5-coder:3b   # 1.9 GB
# 8 GB VRAM — start here if in doubt:
ollama pull qwen2.5-coder:7b     # 4.5 GB
# 16 GB VRAM:
# ollama pull qwen2.5-coder:14b  # 9 GB
# 24 GB+ VRAM only:
# ollama pull qwen2.5-coder:32b  # 19 GB — will black-screen smaller cards
```

### Step 3: Install Rust (needed to build claw-code)

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source ~/.cargo/env
```

### Step 4: Build claw-code

```bash
git clone https://github.com/ultraworkers/claw-code.git ~/claw-code
cd ~/claw-code/rust
cargo build --workspace --release
sudo cp target/release/claw /usr/local/bin/claw
claw --version   # verify
```

### Step 5: Fix Ollama systemd config — pick the profile that matches your hardware

Default Ollama systemd config has `WatchdogSec=120` in some guides — **remove it or Ollama will die every 10 minutes.** `WatchdogSec` is a two-way heartbeat contract and Ollama doesn't implement the protocol.

Edit `/etc/systemd/system/ollama.service` to match **your** hardware profile below.

#### Profile A — single consumer GPU (8–16 GB, also drives your display)

```ini
[Unit]
Description=Ollama Service
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/ollama serve
User=ollama
Group=ollama
Restart=always
RestartSec=5
Environment="OLLAMA_HOST=127.0.0.1"
Environment="OLLAMA_LOAD_TIMEOUT=5m"
Environment="OLLAMA_NUM_PARALLEL=1"
Environment="OLLAMA_MAX_LOADED_MODELS=1"
Environment="OLLAMA_KEEP_ALIVE=5m"
Environment="OLLAMA_FLASH_ATTENTION=true"
Environment="OLLAMA_KV_CACHE_TYPE=q8_0"
Environment="OLLAMA_GPU_OVERHEAD=2147483648"
TimeoutStartSec=300
# No OLLAMA_SCHED_SPREAD — only one GPU
# No WatchdogSec — see note above
# If you have multiple NVIDIA cards and want to reserve GPU 0 for display:
# Environment="CUDA_VISIBLE_DEVICES=1"

[Install]
WantedBy=default.target
```

Key differences on single-GPU (most important first):
- **`OLLAMA_GPU_OVERHEAD=2147483648`** — reserves 2 GiB of VRAM for your desktop compositor. Single biggest safety setting when the GPU also drives your monitor. Added in Ollama v0.20.
- **`OLLAMA_NUM_PARALLEL=1`** — one request at a time, one KV-cache block. `4` is a multi-GPU setting; using it on a single card is what crashed our friend's 3070 Ti running the 14b.
- **`OLLAMA_MAX_LOADED_MODELS=1`** — prevents a stray `ollama run <other-model>` from co-loading a second model onto your one card.
- **`OLLAMA_KV_CACHE_TYPE=q8_0`** — quantized KV cache, cuts KV memory 2–4x at a small quality cost. Makes 14b comfortable on 12 GB. Leave at default `f16` if you have headroom.
- **`OLLAMA_KEEP_ALIVE=5m`** — VRAM frees up shortly after you stop using it, so the display stays responsive while you work in a browser/editor.
- **Bound to `127.0.0.1`** instead of `0.0.0.0` — nothing on the LAN can hit your Ollama by accident.

Drop `OLLAMA_GPU_OVERHEAD` to `1073741824` (1 GiB) on 4–6 GB cards where 2 GiB of overhead would leave no room for the model.

#### Profile B — dedicated multi-GPU rig (no display on the GPUs)

```ini
[Unit]
Description=Ollama Service
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/ollama serve
User=ollama
Group=ollama
Restart=always
RestartSec=5
Environment="OLLAMA_HOST=0.0.0.0"
Environment="OLLAMA_LOAD_TIMEOUT=10m"
Environment="OLLAMA_SCHED_SPREAD=true"
Environment="OLLAMA_NUM_PARALLEL=4"
Environment="OLLAMA_KEEP_ALIVE=24h"
Environment="OLLAMA_FLASH_ATTENTION=true"
TimeoutStartSec=300
# DO NOT add WatchdogSec — it will kill Ollama every N seconds

[Install]
WantedBy=default.target
```

Then:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

### Step 6: Create the `ccode` launcher

Takes the model from the `CCODE_MODEL` env var (or its first argument), so you can't accidentally launch the 32b on a small GPU. Default is the safe `qwen2.5-coder:7b`.

```bash
sudo tee /usr/local/bin/ccode > /dev/null << 'SCRIPT'
#!/bin/bash
# Local Claude Code launcher
# Pick your model with:  export CCODE_MODEL=qwen2.5-coder:14b
# Or pass it inline:     ccode --model-override qwen2.5-coder:14b
MODEL="${CCODE_MODEL:-qwen2.5-coder:7b}"

# Allow one-shot override: first arg may be --model-override <name>
if [ "$1" = "--model-override" ] && [ -n "$2" ]; then
  MODEL="$2"
  shift 2
fi

mkdir -p "$HOME/claw-test"
cd "$HOME/claw-test" || exit 1
export OPENAI_BASE_URL="http://127.0.0.1:11434/v1"
export OPENAI_API_KEY="ollama"
exec claw --model "openai/$MODEL" "$@"
SCRIPT
sudo chmod +x /usr/local/bin/ccode
```

To make your chosen model permanent, add to `~/.bashrc` (or `~/.zshrc`):

```bash
export CCODE_MODEL=qwen2.5-coder:14b   # use whatever matches your VRAM tier
```

### Step 7: Smoke-test the plain model BEFORE using `ccode`

This takes 20 seconds and tells you whether your Profile A config actually holds before you involve claw-code at all. If this step freezes the display, `ccode` would have crashed the box — better to find out with plain Ollama where `Ctrl+C` actually works.

```bash
# pick the size you pulled in Step 2
ollama run qwen2.5-coder:7b "reply with the word OK"
```

Expected: model loads (first time: 20–60s), prints OK, returns to prompt. Your mouse cursor should stay responsive the whole time.

**If the smoke test hangs or the display stutters badly:** drop one tier (`:7b` → `:3b`, or `:3b` → `:1.5b`), or switch to a tighter quantization (`qwen2.5-coder:7b-instruct-q4_K_M`), or set `OLLAMA_GPU_OVERHEAD` higher.

---

## Running it

```bash
ccode
```

That's it. First launch takes 30–60 seconds while the model loads into VRAM. After that it's fast.

You'll see a `>` prompt. Type anything:

```
> read config.py and explain what it does
> write a bash script that pings 10 hosts in parallel
> grep for all functions named "login" in this repo
> /doctor    # built-in health check
> /help      # list commands
```

---

## Gotchas we hit (so you don't have to)

### 1. The `WatchdogSec` trap
See step 5 above. **Never set `WatchdogSec` on the Ollama systemd service.** Ollama doesn't implement systemd's watchdog protocol, so systemd kills it at the watchdog interval. We spent 2 days watching 17 GB model downloads fail at the 10-minute mark before spotting this.

### 2. `claw` refuses to run in your home directory
claw-code considers `$HOME` "too broad" and won't launch there for safety. Always cd into a subdirectory first (the `ccode` wrapper handles this).

### 3. Gemma 3 doesn't support tools in Ollama
`gemma3:27b` is fast (~20 tok/s vs qwen3:32b's 3 tok/s) but Ollama reports it as non-tool-capable, so claw-code rejects it. **Use `qwen2.5-coder` (any size in your tier) for agent work, reserve gemma3 for plain chat.**

### 4. qwen3 has "thinking mode"
`qwen3:32b` generates hundreds of hidden reasoning tokens before every response. Smarter but ~80% slower. The `qwen2.5-coder` family (any size) doesn't do this — use it unless you specifically want the extra reasoning.

### 5. Detached terminal output blocks hashcat-like tools
If you run `claw` (or hashcat, etc.) inside a detached `screen` session, output to the terminal can trigger SIGTTOU and stop the process. Always redirect output to a file when running detached.

### 6. Restrict which GPUs are visible
If you want claw-code to only use certain GPUs (e.g., leave one free for display), set `CUDA_VISIBLE_DEVICES=0,1,2,3` in the Ollama service environment. Without it, Ollama grabs all GPUs.

### 7. Co-resident tools — GPU-touching vs CPU-only
Two categories of "is it safe to run alongside `ccode`?":

- **Safe (CPU/network only, zero VRAM contention):** Go/Python/Bash CLIs that don't touch CUDA. Examples: HTTP scanners, nmap, custom OSINT collectors, tmux, htop. A fast Go liveness scanner making thousands of concurrent requests can max a CPU core but won't take a byte of VRAM.
- **Dangerous on single-GPU rigs (compete for VRAM):** anything that loads a model or uses CUDA directly. A second `ollama run`, `llama.cpp` (`llama-cli`), LM Studio CLI, koboldcpp, Stable Diffusion, CUDA-accelerated video encoders. On an 8 GB card, two model-loading tools simultaneously is a guaranteed crash.
- **Check what's actually holding VRAM:** `nvidia-smi --query-compute-apps=pid,process_name,used_memory --format=csv` lists every process currently reserving VRAM — including llama.cpp-based tools that bypass Ollama entirely and would be invisible to `curl http://127.0.0.1:11434/api/ps`.

---

## Case study: the 3070 Ti black-screen (2026-04-17)

Real incident that motivated most of this guide's safety work.

**Hardware:** single RTX 3070 Ti (8 GB, driving the monitor). Linux, display-shared GPU.

**What they did:** followed an earlier version of this guide linearly, ran `ollama pull qwen2.5-coder:14b`, used the multi-GPU systemd profile verbatim (`OLLAMA_NUM_PARALLEL=4`, `KEEP_ALIVE=24h`, `SCHED_SPREAD=true`), ran `ccode`.

**What happened:** model loaded, display went fully black. SSH-reachable from another machine, but compositor dead. Hard reboot required. Second attempt: same crash.

**Post-mortem VRAM math for the 14b they were running:**

```
Weights (14b, f16):        9.0 GB
KV cache × 4 (NUM_PARALLEL):  ~6.0 GB     ← the killer
Desktop/compositor:         ~1.5–2.0 GB
                          ─────────────
Total needed:              ~16.5–17 GB
Actual VRAM available:        8.0 GB
                          ─────────────
Over budget by:            ~8–9 GB
```

Ollama spilled layers to CPU / unified memory. NVIDIA kernel driver had no spare framebuffer to give the compositor. Display output dropped. Linux was still alive underneath — hence the "SSH worked but screen was black" symptom.

**What fixed it:**
- Dropped to `qwen2.5-coder:7b` (4.5 GB weights fits 8 GB card with margin).
- Set `OLLAMA_NUM_PARALLEL=1` — only one KV-cache block.
- Added `OLLAMA_GPU_OVERHEAD=2147483648` to reserve 2 GiB for the compositor.
- Added `OLLAMA_MAX_LOADED_MODELS=1` in case a second model loaded.
- Smoke-tested with plain `ollama run` before launching `ccode`.

**False suspect (worth naming):** a small Go HTTP scanner called Loki was running in another terminal at crash time. Loki is pure CPU/network, never touches CUDA, allocated zero VRAM — confirmed via `nvidia-smi --query-compute-apps`. If you ever see a crash and a background tool gets blamed: check the VRAM occupancy list *before* drawing conclusions. The background tool is almost never the real cause.

---

## How it compares to real Claude Code

| | Local (this setup) | Real Claude Code |
|---|---|---|
| **Cost** | $0 per token, ~$0.15/hr electricity at full GPU load | ~$3–15 per million tokens |
| **Privacy** | 100% local, nothing leaves your box | Sent to Anthropic |
| **Quality** | Good for routine coding | Best-in-class reasoning |
| **Speed** | ~15 tok/s on a 32B model | ~50–100 tok/s |
| **Context window** | ~32K usable | 200K–1M |
| **Tool use** | Works, occasionally flaky | Excellent |
| **Web search / sub-agents / MCP** | Not yet | Yes |

**Smart play:** use local for anything private (client code, credentials, internal stuff) and anything routine. Use real Claude for heavy architectural reasoning, large-context work, and anything where quality matters more than tokens.

---

## Links

- **[ultraworkers/claw-code](https://github.com/ultraworkers/claw-code)** — the Rust harness (what we run)
- **[instructkr/claude-code](https://github.com/instructkr/claude-code)** — the Python sibling (same fork network, same community)
- **[Ollama](https://ollama.ai)** — the model server
- **[qwen2.5-coder model card](https://ollama.com/library/qwen2.5-coder)** — the model we use

---

## TL;DR commands

**First, check the VRAM table at the top of this guide and pick a model your GPU can handle.** The commands below default to the safe `qwen2.5-coder:7b`.

```bash
# Install
curl -fsSL https://ollama.ai/install.sh | sh
ollama pull qwen2.5-coder:7b      # change to :14b or :32b ONLY if your VRAM supports it
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y && source ~/.cargo/env
git clone https://github.com/ultraworkers/claw-code.git ~/claw-code
cd ~/claw-code/rust && cargo build --workspace --release
sudo cp target/release/claw /usr/local/bin/claw

# Launcher (creates the 'ccode' command — model is parameterized via CCODE_MODEL)
sudo tee /usr/local/bin/ccode > /dev/null << 'SCRIPT'
#!/bin/bash
MODEL="${CCODE_MODEL:-qwen2.5-coder:7b}"
if [ "$1" = "--model-override" ] && [ -n "$2" ]; then MODEL="$2"; shift 2; fi
mkdir -p "$HOME/claw-test" && cd "$HOME/claw-test" || exit 1
export OPENAI_BASE_URL="http://127.0.0.1:11434/v1"
export OPENAI_API_KEY="ollama"
exec claw --model "openai/$MODEL" "$@"
SCRIPT
sudo chmod +x /usr/local/bin/ccode

# Don't forget to apply Profile A (single-GPU) or Profile B (multi-GPU) from Step 5
# to /etc/systemd/system/ollama.service before running ccode.

# Run
ccode
```

---

*Setup walkthrough based on a working deployment on 5x RTX 3070 rig (40 GB VRAM, Kali Linux, April 2026).*

---

## Emergency: PC black-screened or hung after launching `ccode`

Almost always one thing: the model didn't fit in VRAM, the NVIDIA driver hung, and the compositor died with it. Hold the power button, reboot, then before doing anything else:

```bash
# Stop Ollama immediately and keep it from auto-starting on the next boot
sudo systemctl stop ollama
sudo systemctl disable ollama

# Remove the oversized model so you can't accidentally load it again
ollama rm qwen2.5-coder:32b 2>/dev/null
# (harmless if it wasn't installed)

# Pull a safe model for your VRAM (see the table at the top)
ollama pull qwen2.5-coder:7b    # safe on any 8 GB+ GPU

# Fix your /etc/systemd/system/ollama.service to Profile A (single-GPU)
# — especially OLLAMA_NUM_PARALLEL=1 and OLLAMA_KEEP_ALIVE=5m
sudo systemctl daemon-reload
sudo systemctl enable ollama
sudo systemctl start ollama

# Verify you can chat with the small model in plain Ollama BEFORE touching ccode
ollama run qwen2.5-coder:7b "say hello"
```

Only launch `ccode` again once plain `ollama run` works cleanly. If `ollama run` itself hangs the display, your GPU is being overcommitted — use a smaller quantization (`qwen2.5-coder:7b-instruct-q4_K_M`) or run Ollama on CPU-only by setting `CUDA_VISIBLE_DEVICES=""` in the service file while you investigate.

### Signs you're overcommitting VRAM

- Mouse cursor freezes for seconds at a time, then jumps
- Screen goes black but the PC is still running (SSH in from another machine to check)
- `nvidia-smi` shows VRAM at 99% with `ollama` as the top process
- `dmesg | grep -i nvidia` shows "fallen off the bus" or XID errors

Any of these = stop, switch to a smaller model, and re-read the VRAM table.
