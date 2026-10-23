# Strata on RTX 3060 12 GB + 32 GB RAM

Tested optimal config for this exact hardware (verified 2026-10-21, engine **0.1.47**, app **0.2.2**). Model: **Coder IQ1_M** — the only size 32 GB RAM fits, and the fastest-reading one (91% of full model on SWE-bench). Update procedure: see `QUICK_UPDATE.md`.

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

Engine **0.1.47** (app 0.2.2), 2026-10-21. Timings live in the `timings` block of the chat
response, not under `usage`.

| Workload                          | Speed                    |
| --------------------------------- | ------------------------ |
| Short replies (decode)            | ~32–44 tok/s             |
| Small-prompt prefill (~5–62 tok)  | ~46–64 tok/s             |
| ~1.7K-token prompt prefill        | ~472–639 tok/s           |
| 12.7K-token prompt prefill        | **854 tok/s** (cold, 0.1.45) |
| Repeat turn, cached conversation  | 12763 of 12781 tokens reused |

### Measured A/B: engine 0.1.45 vs 0.1.47 on this box

Two interleaved rounds (swap the engine, restart, warm up, measure; ~22–23 GiB free RAM at
every engine start so the resident pool is comparable). Each request carries a unique suffix
so the prompt cache cannot serve it — every run reads the prompt for real. The prompt is real
prose from `docs\DETAILS.md`, not repeated filler, so it behaves like an agent conversation.

**Decode (tok/s) — 0.1.47 won 6 of 6 paired comparisons:**

| Context | 0.1.45 | 0.1.47 | Gain |
| ------- | ------ | ------ | ---- |
| 1.7K    | 28.9   | 31.0   | **+7.4%** |
| 6K      | 26.6   | 29.6   | **+11.5%** |
| 13K     | 25.2   | 27.2   | **+7.9%** |

Per round: 1.7K +7.3% / +7.6%, 6K +5.3% / +18.3%, 13K +15.0% / +1.5%. Six wins out of six is
a real signal, and it matches upstream's own measurement on an RTX 3060 12 GB:
**60.36 → 66.52 tok/s (+10.2%, 8 of 8 pairs), same answers** (identity gate identical on 10
prompts, exact checks at 4K and 20K).

**Prefill: parity.** 1.7K +0.5%, 6K -2.5%, 13K +3.3% — inside the noise.

The gain is `STRATA_SM86_PERF`, a table of nine switches the engine now turns on by itself on
sm_86. The start-up log names them:

```
strata: sm_86 (Ampere): 9 exact speed switches on by default (STRATA_SM86_PERF=0 turns them
off; a switch you set is kept): ADAPT_NOWAIT ADAPT_LAG CPU_IQ_ALL_EXACT1 SIDE_MIXER
ATTN_DAILY GR_UP75 GU_SWIGLU QSA_ROWS B6_MMVQ_ROWS
```

`STRATA_SM86_PERF=0` in the config's `env` block turns the whole table off.

The gain is largest at 6K and grows with context, which fits the switch list: `ADAPT_NOWAIT`
and `ADAPT_LAG` change how the adaptive tier keeps the GPU cache fed, and that matters most
once the context is long enough to miss many experts.

### Known issue: the VRAM reserve is now too low (not a 0.1.47 regression)

The engine logs this at the start of some requests:

```
WARNING: only 514 MiB of VRAM free at the start of this request (below 1024 MiB): when the
card is this full the driver can page GPU memory to system RAM, decode falls to a few tok/s
and a window may never finish (#2065). Add --vram-reserve-mib 1664 to the config's args, or
lower --max-context / --batch
```

Counted over every session in `strata-coder-iq1_m.log`:

| Engine | Requests | VRAM-low warnings |
| ------ | -------- | ----------------- |
| 0.1.43 | 370      | 0                 |
| 0.1.45 | 275      | 22 (8%)           |
| 0.1.47 | 193      | 15 (7.8%)         |

**0.1.45 introduced this, not 0.1.47** — 0.1.43 never logged it, and 0.1.45 and 0.1.47 are
the same rate. The reserve of 999 MiB was calibrated against 0.1.43, where the engine's own
cache sizing left more VRAM over. Upstream now recommends **1664** for a card like this.

The trade-off is real: a bigger reserve means fewer expert slots in the GPU cache (the
reserve is subtracted before the cache is sized), so it costs some decode speed and buys
headroom against driver paging. This is worth trying with a real agent session running and
watching whether the warning appears and whether decode collapses; it is not changed in the
config yet because 999 was a deliberate calibration.

### The MTP draft layer, and why `flash-next-mtp` cannot be used

The config's `--mtp F:\STRATA\Strata-data\mtp\rt` is the **draft layer** that makes speculative
decoding work. Its provenance:

1. `tools/mtp_fetch.py` pulls the 31 `mtp.*` tensors out of the **BF16 checkpoint**.
2. `tools/mtp_pack.py --experts q2_0` quantizes the routed experts into `mtp-q2_0.gguf`.
3. `tools/mtp_rt.py` rewrites that into the runtime folder the engine reads: `dense.bin` +
   `experts.bin` + `dense.txt` (a one-line-per-tensor manifest with offsets).

Ours: `mtp-q2_0.gguf` is 847.8 MiB, 512 routed experts at **Q2_0**; at load the engine reports
`draft layer loaded, 878 MiB of VRAM (experts 675, dense 111)` and a `draft head over 40525
tokens` — 40525 ids is the English/code subset, which is what `draft_vocab: en` selects.

**`Strata-data\models\coder-IQ1_M\flash-next-mtp` contains no usable MTP.** The two files are
the *same* draft block in a different lineage:

| | ours (`mtp-q2_0.gguf`) | flash-next-mtp |
| --- | --- | --- |
| architecture tag | `qwen4exp-mtp` | `qwen4exp` |
| tensor names | `mtp.layers.0.*`, `mtp.*` | `blk.48.*` (llama.cpp naming) |
| routed experts | `mtp.layers.0.mlp.experts.*` | `blk.48.ffn_{gate,up}_exps` (Q4_K), `ffn_down_exps` (Q8_0) |

Proof, not inference: `tools\mtp_rt.py` on
`mtp-Qwen3.8-Flash-Next-Q4_K_M.gguf` fails immediately with
`KeyError: 'mtp.layers.0.mlp.experts.gate_up_proj'`. The engine reads **only** the `rt`
folder (`mtp: cannot open <dir>/dense.txt`), and its MTP expert blob is Q2_0 by construction —
18 bytes per 64 values, the same layout as the main model's experts, which is what lets the
same VNNI/GPU kernels serve it. There is no path in 0.2.2 that accepts a `blk.48.*` Q4_K MTP.

Re-packing those files would also be pointless: `mtp_pack.py` builds q2_0 with a **per-block
MSE scale search** from the BF16 checkpoint (the lossless source), so our q2_0 is already the
best q2_0 obtainable. Feeding it an already-lossy Q4_K MTP could only give an equal or worse
q2_0, in the same format — no speed to gain.

### `--mtp-q4` (opt-in): works, no demonstrated gain — left off

0.1.47 supports `--mtp-q4 all|proj|head`, which keeps the draft layer's projections and/or
draft head as Q4_0 copies "fewer bytes per draft step". It does what it says:

```
strata mtp: --mtp-q4: draft head over 40525 tokens as Q4_0 (55.7 MiB read per step, was 81.2)
strata mtp: --mtp-q4: 10 projections as Q4_0 (37.3 MiB)
strata mtp: draft layer loaded, 915 MiB of VRAM (experts 675, dense 111)
```

But it did **not** produce a measurable decode gain. Draft acceptance on matched runs stayed
inside the baseline's own spread: baseline sessions on the same synthetic workload ranged
**66.7% – 87.9%**, and the `--mtp-q4 all` session measured **75.0%**. One session of 14 windows
is not enough to resolve a few percent on this box, and the change is opt-in and off by
default, so it is left off. Test configs are regenerable with
`F:\STRATA\bench\make_mtp_configs.py`.

### Measurement trap: the resident pool, not the flag

The engine's resident RAM pool is sized from MemAvailable at start and swung **7.98 – 16.25 GiB**
across otherwise identical restarts, which moves decode speed by roughly ±30%. That is larger
than any effect measured here, so **comparing two arms across different pools is invalid** — an
early pass of this work reported a "10-point acceptance drop" for `--mtp-q4` that turned out to
be an artifact of comparing one synthetic session against older real-traffic sessions. Always
record the pool from `resident RAM mode: ... GiB of experts in RAM` and only compare runs whose
pools match; `F:\STRATA\bench\mtp_stats.py` prints per-session acceptance, pool and decode from
the log so the comparison is visible rather than assumed.



The same prompt twice on the **same** engine gives different text (the adaptive expert tier
moves experts between VRAM and RAM, so decode is not run-to-run deterministic here). Measured:
two identical greedy requests on 0.1.45 differed on 2 of 3 prompts. So a text difference
between 0.1.45 and 0.1.47 says nothing about the update; upstream's own bitwise identity gate
covers that.

### Long prompts (30K+) — older engine sessions, real agent traffic

Realistic baselines from earlier sessions in `strata-coder-iq1_m.log`, carried forward because no
session has yet run a long conversation on 0.1.47:

- 58–66K-token prompt: **854–893 tok/s** cold prefill (0 reused), on 0.1.43.
- 13–59K context, agent turn: **28–41 tok/s** decode, 512–12442 tokens per reply, on 0.1.43.
- Repeat turns on a cached conversation: prompt reuse is near-total (e.g. 68921 of 69487 tokens),
  so the incremental prefill per turn stays small.

Re-measure these on 0.1.47 after a long coding session has run through it.

### Engine defaults worth knowing (0.1.45 → 0.1.47)

- `STRATA_SM86_PERF` (0.2.0+, on by default on sm_86): the nine-switch decode table above.
  `=0` turns it off. **This is the reason to take 0.1.47 on this card.**
- `STRATA_SSD_KEEPALIVE=1` (0.1.45, on by default): pings the SSD every 100 ms for 60 s after a
  request so it never spins down mid-stream — the right call for expert streaming. `=0` reverts.
- `STRATA_ROUTE_TAIL_SKIP=7` (default since 0.1.43): +10–20% decode on 12–16 GB cards
  that miss experts; answers differ slightly from 0.1.41. `=0` for strict parity.
- `STRATA_DF_QB_Q4=1` (new, on by default): a q4_0 KV cache takes the batched verify window —
  a 6-token graph of 1950 nodes instead of 2610, identical token ids.
- CPU prefill share is still correctly declined on this box (10.1 GiB of file pages vs 4.5 GiB RAM
  free). Do not force it with `=auto`.

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
