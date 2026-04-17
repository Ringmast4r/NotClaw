# artofwar — Running Inventory

Living list of everything running on the artofwar box (10.0.0.30). Update whenever services are added, removed, or the box starts misbehaving.

**Last updated:** 2026-04-17 — initial seed from memory, runtime sections still need paste-in.

---

## Hardware

- **GPUs:** 5x RTX 3070 (8 GB each, 40 GB VRAM total)
- **Storage:** Primary boot drive + `/mnt/wordlists` (500 GB USB HDD, from hash-cracking engagement — still mounted)
- **Network:** 10.0.0.30 on LAN

---

## Known services / workloads

### 1. Ollama — LLM inference server
- **Unit:** `ollama.service` (systemd)
- **API:** `127.0.0.1:11434` (bound `0.0.0.0` per `OLLAMA_HOST`)
- **Version:** 0.18.0 (at time of setup)
- **VRAM behavior:** `OLLAMA_SCHED_SPREAD=true` — spreads any loaded model across all 5 GPUs
- **Eviction:** `OLLAMA_KEEP_ALIVE=24h` — models stay resident all day
- **Concurrency:** `OLLAMA_NUM_PARALLEL=4`
- **Config gotcha:** NO `WatchdogSec` (Ollama doesn't implement sd_notify — would be killed every interval)

### 2. claw-code — Rust agent harness
- **Binary:** `/usr/local/bin/claw`
- **Launcher:** `/usr/local/bin/ccode` (cd's into `~/claw-test`, then execs claw)
- **Source:** `~/claw-code/` (clone of `ultraworkers/claw-code`)
- **Resource footprint:** tiny — HTTP client to Ollama, no local inference

### 3. Models on disk (Ollama store)
| Model | Size | Status |
|-------|------|--------|
| `qwen2.5-coder:32b` | 19 GB | Default for `ccode`, tool-capable |
| `qwen3:32b` | 20 GB | Smartest but thinking-mode = slow |
| `qwen3:4b` / `qwen3:1.7b` | 2.5 / 1.4 GB | Test models |
| `gemma3:27b` | 17 GB | Fast chat, no tools |
| `gemma3-tools:27b` | 17 GB | Failed tool-template experiment |

---

## Runtime snapshot (PASTE-IN — refresh when box is sluggish)

Run this on artofwar and paste the output below:

```bash
echo "=== uptime ==="; uptime
echo "=== memory ==="; free -h
echo "=== top CPU ==="; ps -eo pid,pcpu,pmem,rss,comm --sort=-pcpu | head -10
echo "=== top RAM ==="; ps -eo pid,pcpu,pmem,rss,comm --sort=-rss | head -10
echo "=== disk ==="; df -h
echo "=== ollama ==="; systemctl is-active ollama
echo "=== models loaded in VRAM ==="; curl -s http://127.0.0.1:11434/api/ps
echo "=== GPU ==="; nvidia-smi
echo "=== systemd services (running) ==="; systemctl list-units --type=service --state=running --no-pager
echo "=== listening ports ==="; ss -tlnp 2>/dev/null
echo "=== cron jobs ==="; for u in $(cut -f1 -d: /etc/passwd); do crontab -u "$u" -l 2>/dev/null | grep -v '^#' | grep . && echo "  ^ crontab for $u"; done
```

### Last snapshot
_(paste output here — timestamp it)_

**2026-04-17 HH:MM**
```
[paste]
```

---

## Suspected causes of slowness (to check when it happens)

- [ ] Another GPU workload competing (hashcat, second Ollama model, training)
- [ ] `qwen2.5-coder:32b` cold-loading (first request after idle can be 60–90 s)
- [ ] GPU thermal throttling (any 3070 > 85 °C stalls the spread scheduler)
- [ ] Disk I/O wait (big `vmstat wa` column → something hammering disk)
- [ ] Low free RAM forcing swap
- [ ] Zombie claw / ollama-runner process from a prior hung session
- [ ] `/mnt/wordlists` HDD spinning up (slow USB, can block processes that touch it)

---

## Things to add when we learn about them

- [ ] Hashcat (active? paused? session name?)
- [ ] Any always-on scanners or collectors
- [ ] Tor / VPN clients
- [ ] Docker containers (`docker ps` if installed)
- [ ] Reverse tunnels (Cloudflare, Tailscale, SSH -R)
- [ ] Scheduled jobs on root crontab
- [ ] Anything auto-started at boot we forgot about

---

## Quick actions

| Want to... | Command |
|---|---|
| Free all GPU VRAM now | `sudo systemctl stop ollama` (will cold-load next ccode) |
| See who's on the GPUs | `nvidia-smi` (check Processes section) |
| Kill a stuck model load | `curl -X POST http://127.0.0.1:11434/api/generate -d '{"model":"qwen2.5-coder:32b","keep_alive":0}'` |
| Watch Ollama live | `journalctl -u ollama -f` |
| Exit a hung claw | Ctrl+C (twice if needed) |
