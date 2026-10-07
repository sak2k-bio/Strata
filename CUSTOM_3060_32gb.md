# Strata on RTX 3060 12 GB + 32 GB RAM

<<<<<<< Updated upstream
Tested optimal config for this exact hardware (verified 2026-10-04). Model: **Coder IQ1_M** — the only size 32 GB RAM fits, and the fastest-reading one (91% of full model on SWE-bench).
=======
Tested optimal config for this exact hardware (verified 2026-10-07, engine **0.1.40**). Model: **Coder IQ1_M** — the only size 32 GB RAM fits, and the fastest-reading one (91% of full model on SWE-bench). Updates: see `QUICK_UPDATE.md`.
>>>>>>> Stashed changes

TO RUN THE SERVER:
cd /d "F:\STRATA\Strata"

"F:\STRATA\Strata\.venv\Scripts\python.exe" "F:\STRATA\Strata\serve\server.py" "--engine" "strata" "--config" "F:\STRATA\Strata\strata-coder-iq1_m.json" "--host" "0.0.0.0" "--port" "4444" "--api-key" "xyz"

## Settings that matter (already applied)

| Setting                   | Value                        | Why                                                                                                                                                                                         |
| ------------------------- | ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| model                     | Coder IQ1_M (58 GB download) | only fit at 32 GB RAM; fastest size                                                                                                                                                         |
| `--max-context` | `112640` (110K) | big window; prefill past ~16K is slow. Raised from 71680 on 2026-10-06 (+~270 MiB VRAM for q4_0 KV at this model's ~13 attention layers; the auto expert cache absorbs it as ~130 fewer slots) |
<<<<<<< Updated upstream
| `--kv`                    | `q4_0`                       | KV compression: halves KV VRAM (213→542 MiB free), **+44% decode speed** vs int8                                                                                                            |
| `--vram-reserve-mib`      | `999`                        | **required**: leaves ~1 GB VRAM for prompt prefill; without it 30K-token prompts stall and the engine kills itself (issue #29)                                                              |
| `--prefill` | `auto` | 0.1.39: the engine picks the largest chunk (≤8192) whose buffers the expert cache can lend. Engine source measured on a 12 GB card, 32K prompt: 791→973 tok/s at 8192 vs 4096. Was `4096` on 0.1.38; `--vram-reserve-mib 999` stays either way |
=======
| `--kv`                    | `q4_0`                       | KV compression: halves KV VRAM (213→542 MiB free), **+44% decode speed** vs int8. At 112640 context the whole KV (~737 MiB) still fits VRAM, so `--kv-grow` (added 0.1.40, stays in the config) is inert — the engine logs `kv-grow is off` and only activates it if KV stops fitting (262K+ context) |
| `--vram-reserve-mib`      | `999`                        | **required**: leaves ~1 GB VRAM for prompt prefill; without it 30K-token prompts stall and the engine kills itself (issue #29)                                                              |
| `--prefill` | `4096` | fixed prompt chunk; pairs with `--vram-reserve-mib 999` for large-prompt stability (30K+ prompts used to stall past ~16K without it, issue #29). Reverted from `auto` on 2026-10-07 — `auto` gave a slightly larger chunk (6912–7424) but token speed was slightly reduced on this card |
>>>>>>> Stashed changes
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
<<<<<<< Updated upstream
   # {"status":"ok","max_context":71680,...,"loaded":true}
=======
   # {"status":"ok","max_context":112640,...,"loaded":true}
>>>>>>> Stashed changes
   ```

## Connect a client

- **OpenAI-compatible apps**: base URL `http://100.118.62.87:4444/v1` (this machine's Tailscale IP; use `127.0.0.1:4444` locally), API key from the run bat, model name `strata`.
- **Claude Code**: `ANTHROPIC_BASE_URL=http://100.118.62.87:4444`, `ANTHROPIC_MODEL=claude-sonnet-4-5`, any `ANTHROPIC_AUTH_TOKEN`.
- **Browser UI**: `http://127.0.0.1:4444`.

<<<<<<< Updated upstream
## Expected speed (measured, warm)

| Workload                                       | Speed                             |
| ---------------------------------------------- | --------------------------------- |
| Short replies                                  | ~2.9 tok/s                        |
| Long replies (600 tok)                         | ~4.8 tok/s                        |
| Large prompt prefill (20-32K tokens)           | ~200 tok/s cold, ~1000 tok/s warm |
| Tiny prompt prefill (cold, overhead-dominated) | 2–6 tok/s                         |

First requests after a start are slow while the resident expert pool pages in from disk (~1-2 min); everything after that is warm.

## Biggest remaining lever: the data lives on an HDD

`Strata-data` sits on **E:, a mechanical HDD** (WD Blue 1TB SATA), not an SSD — the whole expert-streaming design assumes SSD speeds, and this is why decode sits at ~3-5 tok/s instead of the docs' higher figures. The PC's NVMe (Kingston 1TB) holds C: (28.9 GB free) and F: (22 GB free); Strata-data needs ~85 GB, so moving it needs ~60 GB freed on C: first.

To move once space exists: stop the server, `robocopy E:\Code projects\STRATA\Strata-data F:\Strata-data /E`, update the paths in `strata-coder-iq1_m.json` (all args + `log`), delete the old folder. Expect the largest single speed jump available on this machine.
=======
## Expected speed (measured)

| Workload | Speed |
| --- | --- |
| Short replies (68-token prompt, 200-token cap, engine 0.1.40) | ~27 tok/s decode, ~25 tok/s prompt (58/68 drafts accepted) |
| Long replies (600 tok) | ~4.8 tok/s |
| Large prompt prefill (20-32K tokens) | ~200 tok/s cold, ~1000 tok/s warm |
| Tiny prompt prefill (cold, overhead-dominated) | 2–6 tok/s |

First requests after a start are slow while the resident expert pool pages in from disk (~1-2 min); everything after that is warm. Decode on tiny prompts improved 0.1.39 → 0.1.40 (#783 kernel batch: sub-warp IQ expert kernels, batched K/V for the verify window, MTP drafter catch-up) — the ~27 tok/s figure is one warm request, treat as indicative, not a clean benchmark.

## Data is on the F: NVMe SSD (done 2026-10-05)

`Strata-data` now lives at `F:\STRATA\Strata-data` on the NVMe (was E:, a mechanical
HDD). The move was the largest single speed jump available on this machine — the whole
expert-streaming design assumes SSD reads. Config and run bat were repointed from the
old `E:\Code projects\STRATA\` paths in the same change.
>>>>>>> Stashed changes

## Considered and skipped (on 32 GB RAM)

- **`--conversation-cache-mib`** (park multiple chats in RAM): needs ~2.5 GB+ physical headroom to park at all; the resident expert pool already page-locks ~18 GB. Would almost never trigger.
- **`--experimental-speed-projection`**: changes how the model answers (refusal-direction projection) — not a pure speed win.
- **`--pool-workers`**: calibration measured 6/9/4 workers; the engine's own default won, so nothing is pinned.
- **`--lazy` / `--idle-unload`**: unload the model when idle — the opposite of what we want.
- **Defender exclusion for Strata-data**: would remove scan overhead on expert reads, but weakens security on this PC; do it yourself only if you accept that trade.

## Stays running (no auto-shutdown)

- **Engine watchdog disabled** — `env: {"STRATA_WATCHDOG_S": "0"}` in the config. Trade-off: if the engine ever truly hangs, the server will not recover by itself — close and reopen the run bat. (With the VRAM reserve fix, the stall that used to trip it is gone.)
- **Idle unload off** — the default; the model stays loaded between requests.
- **Windows power plan: High Performance** — Balanced lets clocks drop between requests.
- **Disk never sleeps** — `powercfg /change -disk-timeout-ac 0` (was 20 min); the engine mmap's experts from disk.
- **Sleep/hibernate: never on AC** (already the case; verify with `powercfg /query SCHEME_CURRENT SUB_SLEEP`).
- Keep heavy apps closed before start — free RAM at start = bigger resident expert pool = faster.

## Large prompts (30K+ tokens)

<<<<<<< Updated upstream
Without `--vram-reserve-mib 999` the prefill path runs out of scratch VRAM on prompts past ~16K tokens, stalls at "layer 0 of the prompt chunk", and the engine's 60 s watchdog kills itself (issue #29) — the server then restarts it (~8 min) and the client's retry loops forever. The reserve fixes it (a 32K-token prompt now prefills in ~30 s); with `--prefill auto`, 0.1.39 picks the chunk size within that reserve. If you change `--max-context` or `--kv`, keep the reserve.
=======
Without `--vram-reserve-mib 999` the prefill path runs out of scratch VRAM on prompts past ~16K tokens, stalls at "layer 0 of the prompt chunk", and the engine's 60 s watchdog kills itself (issue #29) — the server then restarts it (~8 min) and the client's retry loops forever. The reserve fixes it (a 32K-token prompt now prefills in ~30 s); `--prefill 4096` keeps the chunk small enough to stay inside that reserve. If you change `--max-context` or `--kv`, keep the reserve.
>>>>>>> Stashed changes

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
<<<<<<< Updated upstream
- `--vram-reserve-mib` → `999` and `--prefill` → `auto` (large-prompt stability, see above)
=======
- `--vram-reserve-mib` → `999` and `--prefill` → `4096` (large-prompt stability, see above)
>>>>>>> Stashed changes
- top-level `"draft_vocab": "en"` and `"reasoning_budget_tokens": 512`
- top-level `"env": {"STRATA_WATCHDOG_S": "0"}` (no auto-shutdown; see trade-off above)

And add to the run script: `--host 0.0.0.0 --port 4444 --api-key <random-48-chars>`.

Then tune for the hardware (5-10 min, writes `--pcie-frac`/`--spec-min-p` into the config, kept across updates):

```
START-HERE.bat --yes --calibrate --no-start
```

And for English/code use, shrink the draft head (`--draft-vocab en`, ~130 MiB less VRAM for the expert cache).

### Known issues fixed in this tree

<<<<<<< Updated upstream
- **`setup.py` disk check** (≈ line 3410): a resumed install re-reserved ~32 GB for `experts.bin` even after it was already built, failing with "not enough free disk space". Patched (`arena_done` check) so only genuinely missing space is required.
=======
- **`setup.py` disk check** (≈ line 3410): a resumed install re-reserved ~32 GB for `experts.bin` even after it was already built, failing with "not enough free disk space". Our local patch is gone — upstream fixed this properly in 0.1.40 (`ed15a3d`, "the disk check counts only what step 6 still writes").
>>>>>>> Stashed changes
- **Empty `third_party/llama.cpp/gguf-py`** (interrupted extract): MTP pack step dies with "gguf-py was not found". Rerunning setup re-fetches llama.cpp automatically — just rerun.

### Security notes

- The API key sits in plaintext in `run-coder-iq1_m.bat`. Rotate it (`--api-key <new>`) before sharing the file or the machine.
- `/health` is unauthenticated by design (status only). Everything under `/v1/*` requires the key.
- Windows Firewall: existing `python.exe` inbound rules cover port 4444 on Private (Tailscale) and Public profiles. On a clean machine, allow inbound TCP 4444 for `python.exe`.
