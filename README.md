<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/Ringmast4r/Ringmast4r/output/github-contribution-grid-snake-dark.svg" />
  <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/Ringmast4r/Ringmast4r/output/github-contribution-grid-snake.svg" />
  <img alt="github contribution grid snake animation" src="https://raw.githubusercontent.com/Ringmast4r/Ringmast4r/output/github-contribution-grid-snake-dark.svg" width="100%" />
</picture>

<div align="center">

# NotClaw

[![Ollama](https://img.shields.io/badge/Ollama-v0.21-00C7B7?style=for-the-badge&logo=ollama&logoColor=white)](https://ollama.ai)
[![claw-code](https://img.shields.io/badge/claw--code-Rust-DEA584?style=for-the-badge&logo=rust&logoColor=white)](https://github.com/ultraworkers/claw-code)
[![Rig](https://img.shields.io/badge/artofwar-5x_RTX_3070-76B900?style=for-the-badge&logo=nvidia&logoColor=white)](ARTOFWAR-RUNNING-INVENTORY.md)
[![Workspace](https://img.shields.io/badge/Workspace-PRIVATE_%7C_NTFS_LOCKED-red?style=for-the-badge)](#workspace-status)

**Private workspace for running Claude Code-style agents 100% locally.**

*Our rig. Our research. Every trap we hit — written down so we don't hit it twice.*

</div>

---

## What this folder is

A private, NTFS-locked workspace holding **our** documentation for running local Claude Code-style agents via [claw-code](https://github.com/ultraworkers/claw-code) + [Ollama](https://ollama.ai). It is **not** a GitHub repo, is **not** synced anywhere, and lives side-by-side with — but intentionally separate from — the `Ringmast4r/gridbasedotnet-CLAUDE` fork (that's a patched LiteLLM-proxy approach; this is the claw-code path we actually use daily).

### Files in this folder

| File | Purpose |
|---|---|
| [`LOCAL-CLAUDE-SETUP.md`](LOCAL-CLAUDE-SETUP.md) | **The deep walkthrough.** VRAM-tier table (3b → 70b), Profile A/B systemd configs, Step 7 smoke test, all 7 gotchas, the 2026-04-17 crash case study, emergency recovery. Start here if setting up a rig. |
| [`ARTOFWAR-RUNNING-INVENTORY.md`](ARTOFWAR-RUNNING-INVENTORY.md) | Living inventory of what's running on the `artofwar` box (10.0.0.30). Paste-in triage block for when inference gets slow. |
| `README.md` | This file — landing page + quick reference. |

---

## The stack

```
  ccode                               ← our launcher script
    └─ claw  (Rust binary)            ← the agent harness (tools, loops, file I/O)
         └─ Ollama API                ← 127.0.0.1:11434
              └─ qwen2.5-coder:<tier> ← 3b / 7b / 14b / 32b depending on VRAM
                   └─ GPU(s)          ← 1 consumer card OR a headless rig
```

- [`ultraworkers/claw-code`](https://github.com/ultraworkers/claw-code) — Rust rewrite of Claude Code's agent harness (~185k stars). Native OpenAI-compat tool calling to Ollama, no proxy.
- [`Ollama`](https://ollama.ai) v0.20+ — hosts the model, handles KV cache, spreads across GPUs.
- [`qwen2.5-coder`](https://ollama.com/library/qwen2.5-coder) family — the only Ollama model family we've found that combines reliable tool use with *no* thinking-mode overhead. Tier picks by VRAM.

No LiteLLM proxy. No Anthropic API. No network calls leave the box.

---

## Hardware we've tested on

### `artofwar` — our main rig (headless, dedicated inference)

- **5x NVIDIA RTX 3070** — 40 GB VRAM total, no display attached.
- Runs `qwen2.5-coder:32b` (~19 GB) spread across all 5 cards via `OLLAMA_SCHED_SPREAD=true`, ~15 tok/s generation.
- Uses **Profile B** systemd config (see LOCAL-CLAUDE-SETUP.md §5).
- Also hosts: hashcat wordlists (500 GB USB HDD at `/mnt/wordlists`), the `ccode` launcher, three Qwen model families on disk. Full runtime map in [`ARTOFWAR-RUNNING-INVENTORY.md`](ARTOFWAR-RUNNING-INVENTORY.md).

### Buddy's PC — single RTX 3070 Ti, 2026-04-17 incident

- **1x RTX 3070 Ti** — 8 GB VRAM, driving the monitor.
- **Black-screened** following an older version of the guide. Running `qwen2.5-coder:14b` (9 GB weights) with `OLLAMA_NUM_PARALLEL=4` required ~17 GB; card had 8 GB. NVIDIA driver hung, compositor died.
- Fixed with: `qwen2.5-coder:7b`, `NUM_PARALLEL=1`, `OLLAMA_GPU_OVERHEAD=2147483648`, pre-launch `ollama run` smoke test.
- This incident drives most of the current safety advice — full post-mortem in LOCAL-CLAUDE-SETUP.md.

---

## Ollama env vars — our cheat sheet

Reference card for `/etc/systemd/system/ollama.service`. Full Profile A/B configs live in [`LOCAL-CLAUDE-SETUP.md`](LOCAL-CLAUDE-SETUP.md).

| Variable | What it does | Single-GPU (Profile A) | Multi-GPU rig (Profile B) |
|---|---|---|---|
| `OLLAMA_GPU_OVERHEAD` | Reserves VRAM for compositor. **Prevents black-screens.** Added v0.20. | `2147483648` (2 GiB) | *unset* |
| `OLLAMA_NUM_PARALLEL` | KV-cache blocks reserved at launch. | `1` | `4` |
| `OLLAMA_MAX_LOADED_MODELS` | Blocks a stray `ollama run` from co-loading. | `1` | *unset* |
| `OLLAMA_KV_CACHE_TYPE` | `q8_0` quantizes KV cache (2–4x smaller). | `q8_0` on tight cards | `f16` (default) |
| `OLLAMA_KEEP_ALIVE` | How long model stays in VRAM after last request. | `5m` | `24h` |
| `OLLAMA_SCHED_SPREAD` | Distribute layers across all GPUs. | *unset* | `true` |
| `OLLAMA_FLASH_ATTENTION` | Faster attention kernels, lower VRAM. | `true` | `true` |
| `OLLAMA_HOST` | Bind interface. | `127.0.0.1` | `0.0.0.0` |
| `OLLAMA_LOAD_TIMEOUT` | Tolerance for large-model load. | `5m` | `10m` |
| `WatchdogSec` | **NEVER SET.** Ollama doesn't implement sd_notify — systemd kills it every interval. Burned us 2 days. | NEVER | NEVER |

---

## Model tier summary

| VRAM | Model | Pull size | Notes |
|:---:|---|---|---|
| 4–6 GB | `qwen2.5-coder:3b` | 1.9 GB | Laptop dGPU, integrated graphics |
| 8 GB | `qwen2.5-coder:7b` | 4.5 GB | Safe on display-driving GPU |
| 12 GB | `qwen2.5-coder:7b` (or `:14b` with `q8_0` KV cache) | 4.5 / 9 GB | Mid-range, `NUM_PARALLEL=1` |
| 16 GB | `qwen2.5-coder:14b` | 9 GB | Comfortable agent work |
| 24 GB | `qwen2.5-coder:32b` | 19 GB | First tier that safely fits the 32b |
| 32 GB+ | `qwen2.5-coder:32b` or `llama3.3:70b` | 19 / 40 GB | Multi-GPU rig (our artofwar lives here) |

Smoke-test only (too small for real agent work): `qwen2.5-coder:1.5b` (986 MB), `qwen2.5-coder:0.5b` (398 MB).

---

## Seven traps we have hit (and won't again)

1. **`WatchdogSec` on Ollama systemd** — kills Ollama every N seconds because Ollama doesn't send sd_notify heartbeats. Cost 2 days of failed 17 GB model downloads before we caught it.
2. **Multi-GPU defaults on single-GPU PC** — `NUM_PARALLEL=4` + large model on an 8 GB display-GPU = instant black-screen. Fixed by Profile A with `OLLAMA_GPU_OVERHEAD`.
3. **Gemma 3 has no tool support in Ollama** — fast (~20 tok/s) but Ollama refuses to pass tool schemas. Chat only.
4. **qwen3 has thinking mode** — generates hidden reasoning tokens before each response, ~80% slower. The `qwen2.5-coder` family doesn't.
5. **`claw` refuses to run in `$HOME`** — considers it "too broad." The `ccode` wrapper cd's into `~/claw-test`.
6. **Ctrl+C doesn't kill a hung claw** — raw-mode TTY swallows SIGINT. From a second session: `pkill -INT claw` → `pkill -TERM` → `pkill -KILL`.
7. **Co-resident tools are mostly innocent** — Go/Python/Bash CLIs don't touch CUDA. Dangerous ones: `llama.cpp` / LM Studio / second `ollama run` / Stable Diffusion. Check with `nvidia-smi --query-compute-apps=pid,process_name,used_memory --format=csv`.

Full detail on every one: [`LOCAL-CLAUDE-SETUP.md`](LOCAL-CLAUDE-SETUP.md).

---

## Pre-launch checklist (new machine)

1. Identify GPU + VRAM. `nvidia-smi --query-gpu=name,memory.total --format=csv`.
2. Pick the model that fits the tier. **Never guess.**
3. Write the right systemd profile (A for single-GPU, B for multi-GPU rig).
4. `sudo systemctl daemon-reload && sudo systemctl restart ollama`
5. **Smoke-test plain Ollama:** `ollama run qwen2.5-coder:<tier> "reply with OK"`. Mouse cursor must stay responsive. If not — drop a tier before launching `ccode`.
6. Install / update `ccode` launcher (parameterized via `CCODE_MODEL` env var).
7. Launch `ccode`, test a trivial prompt, verify no display stutter.

Only then do real work.

---

## Related — but separate

- [`ultraworkers/claw-code`](https://github.com/ultraworkers/claw-code) — the Rust harness we run. 185k stars, Linux-x64 + macOS-arm64 release binaries only, build from source recommended.
- [`instructkr/claude-code`](https://github.com/instructkr/claude-code) — Python sibling in the same fork network. Identical commit stream to claw-code.
- [`Ringmast4r/gridbasedotnet-CLAUDE`](https://github.com/Ringmast4r/gridbasedotnet-CLAUDE) — our fork of a LiteLLM-proxy approach to running the real Claude Code npm CLI against Ollama. **Separate concern**, lives at `Desktop/GITHUBS/gridbasedotnet-CLAUDE/`. Note: upstream `gridbasedotnet/CLAUDE` returns 404 as of 2026-04-17 — our fork is now the canonical git home for that path.
- [Ollama](https://ollama.ai) — the model server. Running v0.21 on artofwar as of 2026-04-17.

---

## Workspace status

- **Location:** `C:\Users\Squir\Desktop\GITHUBS\NotClaw\`
- **NTFS permissions:** Inheritance broken 2026-04-17. Only `DESKTOP-QKOB615\Squir` + `NT AUTHORITY\SYSTEM` have access. `BUILTIN\Administrators` removed.
- **Git:** Not a repo. No remote. Files are ours alone.
- **Ollama minimum for everything here:** v0.20 (for `OLLAMA_GPU_OVERHEAD`). On v0.21 as of 2026-04-17.
- **Latest incident folded in:** 2026-04-17 — single 3070 Ti black-screen case study.

---

*Private workspace — [@Ringmast4r](https://github.com/Ringmast4r) — April 2026.*
