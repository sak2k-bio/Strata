# Strata on RTX 3060 12 GB + 32 GB RAM

Tested optimal config for this exact hardware (verified 2026-10-12, engine **0.1.43**). Model: **Coder IQ1_M** — the only size 32 GB RAM fits, and the fastest-reading one (91% of full model on SWE-bench). Update procedure: see `QUICK_UPDATE.md`.

TO RUN THE SERVER:
cd /d "F:\STRATA\Strata"

"F:\STRATA\Strata\.venv\Scripts\python.exe" "F:\STRATA\Strata\serve\server.py" "--engine" "strata" "--config" "F:\STRATA\Strata\strata-coder-iq1_m.json" "--host" "0.0.0.0" "--port" "4444" "--api-key" "xyz"

## Settings that matter (already applied)

| Setting                   | Value                        | Why                                                                                                                                                                                         |
| ------------------------- | ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| model                     | Coder IQ1_M (58 GB download) | only fit at 32 GB RAM; fastest size                                                                                                                                                         |
| `--max-context` | `112640` (110K) | big window; prefill past ~16K is slow. Raised from 71680 on 2026-10-06 (+~270 MiB VRAM for q4_0 KV at this model's ~13 attention layers; the auto expert cache absorbs it as ~130 fewer slots) |
| `--kv`                    | `q4_0`                       | KV compression: halves KV VRAM (213→542 MiB free), **+44% decode speed** vs int8. `--kv-grow` is in the config but inert at 112640 context (the whole KV ~737 MiB still fits VRAM); the engine logs `kv-grow is off` and only activates it if KV stops fitting (262K+) |
| `--vram-reserve-mib`      | `999`                        | **required**: leaves ~1 GB VRAM for prompt prefill; without it 30K+ prompts stall and the engine kills itself (issue #29)                                                              |
| `--prefill` | `4096` | fixed chunk; stays well inside the `--vram-reserve-mib 999` scratch so 30K+ prompts never stall. Tried `auto` (the engine picks the biggest chunk ≤8192 that fits) — it landed on 6912–7424 and measured slightly *slower* on this card, so back to 4096 on 2026-10-07 |
| `--pcie-frac`             | `0.35`                       | **calibrated on this PC** (setup `--calibrate`): share of missing experts copied over PCIe vs read from SSD                                                                                 |
| `--spec-min-p`            | `0.70`                       | **calibrated**: how sure the draft layer must be to extend a verify window                                                                                                                  |
| `draft_vocab`             | `en`                         | English/code draft head: 81 MiB VRAM vs 213 MiB for the cjk default — frees ~130 MiB for the expert cache                                                                                   |
| `reasoning_budget_tokens` | `512`                        | caps thinking per request; server closes the thinking block at the budget so replies always answer (uncapped default burned the whole `max_tokens` on this slow card — "no answer" replies) |
| `--prompt-cache-every`    | `4096`                       | checkpoint growing prompts every 4K tokens instead of 16K — agent clients (Claude Code) resend the whole conversation each turn, so this is what makes multi-turn work fast                 |
| `fit_max_tokens`          | `true`                       | server clamps `max_tokens` to the context room instead of returning 400 — agent clients that overshoot get a shorter answer, not an error                                                   |
| `--resident-experts`      | on                           | experts the GPU can't hold live in RAM, copied once at start                                                                                                                                |
| `--host` / `--port`       | `0.0.0.0` / `4444`           | reachable from tailnet/LAN                                                                                                                                                                  |
| `--api-key`               | see run bat                  | required — `/v1/*` returns 401 without it                                                                                                                                                   |

## Start the server

1. Close heavy programs first. Free RAM at start = bigger resident expert pool = faster inference.
2. Run `run-coder-iq1_m.bat`.
3. Wait ~5 minutes. First start page-locks 13–17 GB of experts into RAM and the PC may lag briefly — normal.
4. Verify:
   ```
   curl http://127.0.0.1:4444/health
   # {"status":"ok","max_context":112640,...,"loaded":true}
   ```

## Connect a client

- **OpenAI-compatible apps**: base URL `http://100.118.62.87:4444/v1` (this machine's Tailscale IP; use `127.0.0.1:4444` locally), API key from the run bat, model name `strata`.
- **Claude Code**: `ANTHROPIC_BASE_URL=http://100.118.62.87:4444`, `ANTHROPIC_MODEL=claude-sonnet-4-5`, any `ANTHROPIC_AUTH_TOKEN`.
- **Browser UI**: `http://127.0.0.1:4444`.

## Expected speed (measured)

Short prompts, engine 0.1.43, 2026-10-12 (5-run medians from `strata-coder-iq1_m.log`; timings live in the `timings` block of the chat response, not under `usage`):

| Workload                          | Speed                    |
| --------------------------------- | ------------------------ |
| Short replies (decode)            | ~38–43 tok/s             |
| Long replies (600 tok)            | ~31 tok/s                |
| Small-prompt prefill (~5 tok)     | ~44–63 tok/s             |
| ~950-token prompt prefill         | ~368 tok/s               |
| Long prompts (20-66K tokens)      | see below — measured on 0.1.40 |

### Long prompts (30K+) — engine 0.1.40, real agent traffic

These come from the **previous** engine session (the log's 0.1.40 block), not 0.1.43 — the only
session that has carried long conversations. Realistic baseline:

- 66K-token prompt: **758 tok/s** cold prefill (0 reused).
- 66–72K context, agent turn: **20–28 tok/s** decode, 200–740 tokens per reply, 91–93% draft acceptance.
- Repeat turns on a cached conversation: prompt reuse is near-total (e.g. 68921 of 69487 tokens),
  so the incremental prefill per turn stays small.

Re-measure these on 0.1.43 after a long coding session has run through it. The short-prompt table
above is the only thing measured directly on the new engine so far.

## Data location: moved off the HDD (done)

`Strata-data` was on **E:, a mechanical HDD**, which capped decode at ~3–5 tok/s — the expert-streaming design assumes SSD speeds. It now lives on the **F: NVMe** (`F:\STRATA\Strata-data`), and decode sits at ~38–43 tok/s (see the speed table above). The move was the single largest speed gain available on this PC. The old path was `E:\Code projects\STRATA\Strata-data`.

## Considered and skipped (on 32 GB RAM)

- **`--conversation-cache-mib`** (park multiple chats in RAM): needs ~2.5 GB+ physical headroom to park at all; the resident expert pool already page-locks ~18 GB. Would almost never trigger.
- **`--experimental-speed-projection`**: changes how the model answers (refusal-direction projection) — not a pure speed win.
- **`--pool-workers`**: calibration measured 6/9/4 workers; the engine's own default won, so nothing is pinned.
- **`--lazy` / `--idle-unload`**: unload the model when idle — the opposite of what we want.
- **Defender exclusion for Strata-data**: would remove scan overhead on expert reads, but weakens security on this PC; do it yourself only if you accept that trade.
- **CPU prefill share** (`STRATA_PREFILL_CPU_SHARE`): on by default from 0.1.41 for our shape (CUDA, one GPU, no batch slots), and worth −20..−35% prefill on 512–1000-token prompts on an RTX 3060 in upstream's testing. But 0.1.42 added a memory guard and on this 32 GB box the engine **declines it**: it logs `the CPU prefill share is not armed by default: ... only 4.7 GiB of RAM is free`. Expected and correct — do not force it on with `=auto`; upstream saw failed chats and engine restarts on tight-RAM machines.

## Stays running (no auto-shutdown)

- **Engine watchdog disabled** — `env: {"STRATA_WATCHDOG_S": "0"}` in the config. Trade-off: if the engine ever truly hangs, the server will not recover by itself — close and reopen the run bat. (With the VRAM reserve fix, the stall that used to trip it is gone.)
- **Idle unload off** — the default; the model stays loaded between requests.
- **Windows power plan: High Performance** — Balanced lets clocks drop between requests.
- **Disk never sleeps** — `powercfg /change -disk-timeout-ac 0` (was 20 min); the engine mmap's experts from disk.
- **Sleep/hibernate: never on AC** (already the case; verify with `powercfg /query SCHEME_CURRENT SUB_SLEEP`).
- Keep heavy apps closed before start — free RAM at start = bigger resident expert pool = faster.

## Large prompts (30K+ tokens)

Without `--vram-reserve-mib 999` the prefill path runs out of scratch VRAM on prompts past ~16K tokens, stalls at "layer 0 of the prompt chunk", and the engine's 60 s watchdog kills itself (issue #29) — the server then restarts it (~8 min) and the client's retry loops forever. The reserve fixes it (a 32K-token prompt now prefills in ~30 s); `--prefill 4096` keeps the chunk small enough to stay inside that reserve. If you change `--max-context` or `--kv`, keep the reserve.

## Fresh install from scratch

```
git clone https://github.com/Niko1221/Strata
cd Strata
START-HERE.bat --yes --family coder --model IQ1_M --no-start
```

Needs ~80 GB free on one drive, an NVIDIA GPU (RTX 30-series driver or newer), 32 GB RAM. No Hugging Face token needed (model repos are public).

After setup finishes, apply these edits to `strata-coder-iq1_m.json`:

- `--max-context` → `112640`
- `--kv` → `q4_0`
- `--vram-reserve-mib` → `999` and `--prefill` → `4096` (large-prompt stability, see above)
- top-level `"draft_vocab": "en"` and `"reasoning_budget_tokens": 512`
- top-level `"env": {"STRATA_WATCHDOG_S": "0"}` (no auto-shutdown; see trade-off above)

And add to the run script: `--host 0.0.0.0 --port 4444 --api-key <random-48-chars>`.

Then tune for the hardware (5-10 min, writes `--pcie-frac`/`--spec-min-p` into the config, kept across updates):

```
START-HERE.bat --yes --calibrate --no-start
```

And for English/code use, shrink the draft head (`--draft-vocab en`, ~130 MiB less VRAM for the expert cache).

### Known issues fixed in this tree

- **`setup.py` disk check** (≈ line 3410): a resumed install re-reserved ~32 GB for `experts.bin` even after it was already built, failing with "not enough free disk space". The local patch is gone — upstream fixed this properly in 0.1.40 (`ed15a3d`, "the disk check counts only what step 6 still writes").
- **Empty `third_party/llama.cpp/gguf-py`** (interrupted extract): MTP pack step dies with "gguf-py was not found". Rerunning setup re-fetches llama.cpp automatically — just rerun.

### Security notes

- The API key sits in plaintext in `run-coder-iq1_m.bat`. Rotate it (`--api-key <new>`) before sharing the file or the machine.
- `/health` is unauthenticated by design (status only). Everything under `/v1/*` requires the key.
- Windows Firewall: existing `python.exe` inbound rules cover port 4444 on Private (Tailscale) and Public profiles. On a clean machine, allow inbound TCP 4444 for `python.exe`.
