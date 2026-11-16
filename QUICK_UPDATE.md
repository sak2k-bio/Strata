# QUICK_UPDATE.md — safely update Strata (RTX 3060 / 32 GB install)

Verified 2026-11-16: app 0.2.7.1 → 0.2.14 (engine 0.1.52 → 0.1.60), server + engine together,
custom config untouched (branch `update-0.2.14`). Before that 2026-10-29, app 0.2.2 → 0.2.7.1;
2026-10-21, app 0.1.45 → 0.2.2; 2026-10-17, v0.1.43 → v0.1.45; 2026-10-12, v0.1.40.1 → v0.1.43;
2026-10-05, v0.1.38 → v0.1.39.
This install carries hand-tuned pieces: `strata-coder-iq1_m.json` + `run-coder-iq1_m.bat` (both
gitignored — git never touches them), calibrated `--pcie-frac 0.35` / `--spec-min-p 0.70`,
F:\STRATA paths, and port 4444.

## 0.2.14: the first update since 0.2.0 that measurably pays here (2026-11-16)

Nine tags landed between 0.2.7.1 and 0.2.15 (engine 0.1.53 → 0.1.60). Take **0.2.14, not 0.2.15**:
0.2.15 ships the *same* engine files as 0.2.14 (app/Terminal fixes only, and we run the server from
source), so it adds nothing here.

- **`--lookup-max-t 12` is on by default in the serve engine and is worth +25% on agent rewrites**
  (measured, pool-matched; see `CUSTOM_3060_32gb.md`). It proposes long runs of tokens by looking
  them up in the prompt — file rewrites and edit turns, not chat. This is the reason to update.
- **The route prior is off again.** 0.2.7 turned `STRATA_ROUTE_PRIOR=1.5` on by itself on a card
  like this; **0.2.8 took that back** (owner decision 2026-10-29) because it repeats itself and
  gives weaker results in long agent sessions (#4402). This config now pins `STRATA_ROUTE_PRIOR=0`
  in its `env` block. **The update takes it away; do not add it back.**
- **Engine chain:** 0.1.52 (0.2.7) → 0.1.55 (0.2.9) → 0.1.56 (0.2.10) → 0.1.57 (0.2.11) →
  0.1.58 (0.2.12) → 0.1.59 (0.2.13) → **0.1.60 (0.2.14)**. Most of that work is other platforms
  (Strix Halo/gfx12, Metal, SYCL, Mac, DeepSeek, new model shapes). 0.2.9 is where 0.2.8's
  route-prior revert reached the release line.
- **New dependency:** `pillow` 12.3.0 → **13.0.0** (Python ≥ 3.11; this venv is 3.13), which
  carries the fix for GHSA-78g4-xqwv-q3g8 (OpenJPEG heap overflow reachable from
  `Image.open().load()`). Step 6 below is therefore **required**. `truststore` from the 0.2.7
  update is already installed.
- **Opt-ins worth knowing** (all off by default, none measured on this box yet):
  `STRATA_EMB_REUSE_ACCOUNT=1` (prompt +3.9% on an RTX 3060, upstream's number),
  `STRATA_ADAPT_OVERLAP=1` (+4–8% on an A4000/P100, loses on an RTX 5070), `STRATA_MTP_ROOM_MIN`
  (+17–23% prompt on an 8 GB card), and the auto PCIe-share rule that now refines a missing
  `--pcie-frac` from the CPU pool's measured rate (ours is set, and a set value always wins).

### A/B-ing anything here needs the pool pinned

Carried over from the 0.2.7 work and now the standing rule: the engine sizes its resident expert
pool from available RAM at start, and on this box that figure swung 3.65 → 16.25 GiB across
restarts of *identical* builds, moving decode ~2.6×. Set `STRATA_RESIDENT_HEADROOM_GIB` so both
arms land on the same pool (`F:\STRATA\bench\start_matched.ps1` solves for it), and read the
realised pool back from `resident RAM mode: ... GiB of experts in RAM`. The 0.2.14 rewrite
comparison was run that way — and the winning arm had the *smaller* pool, so the +25% is
conservative.

## 0.2.7 line: what is worth taking and what is not (2026-10-29)

Nine tags landed between 0.2.2 and 0.2.7.1. For **this** install (Windows x86, one RTX 3060,
Qwen IQ1_M, Terminal/server, no desktop app in use):

- **0.2.7.1 adds nothing here.** Its own release text: "If you use Windows, Linux with an x86
  processor, or a Mac for Qwen only, you get no new engine, no speed change and no app change,
  and you do not need to update." It is DGX Spark (ARM64) plus DeepSeek on Mac. The Qwen engine
  is **0.1.52** from 0.2.7 onward. Control moved 0.1.52 → 0.1.53, which only matters to the
  installed Terminal, not to this source checkout.
- **Engine chain:** 0.1.47 (0.2.2) → 0.1.48 (0.2.3) → 0.1.50 (0.2.5) → 0.1.52 (0.2.7). 0.2.4
  added DeepSeek on a *separate* engine package (`strata-dsv41`) and left the Qwen engine alone.
- **The headline win is unverified here. See `CUSTOM_3060_32gb.md`:** residency-biased routing
  (`STRATA_ROUTE_PRIOR`) is on by default on this card, but at a matched 10 GiB pool it changed
  no routing counter (GPU hit 71.1% vs 72.8%, PCIe 6.0% vs 5.6%, disk 9,970 vs 9,629 MB/req).
  Take 0.2.7 for its fixes, not its speed.
- **What is genuinely new for us:** the 0.2.7 fix list — a conversation cache no longer reuses
  another conversation's keys/values (#1503), a draft-vocabulary row cannot be written past its
  bounds (#3911), a checkpoint save that runs out of memory no longer exits the engine (#3350),
  and undeclared tool calls returned as text are recovered. The last one matters: agents drive
  this server.
- **New dependency:** `requirements.txt` gained `truststore==0.10.5` (Python ≥ 3.10) and bumped
  `certifi` to 2026.10.19 — HTTPS now uses the OS certificate store (#3722). Step 6 below is
  therefore **required**, not optional, for this update.

### Pool sizing is now controllable — use it for any A/B

`STRATA_RESIDENT_HEADROOM_GIB` (GiB of available RAM left free at start; default 4) makes the
resident expert pool deterministic: `pool = available RAM at start − headroom − 0.25`. Without
it, the pool swung 3.65 → 16.25 GiB across restarts of identical builds and moved decode ~2.6×,
which invalidated every earlier comparison. `F:\STRATA\bench\start_matched.ps1` solves for the
headroom that lands an arm on a target pool; `route_stats.py` / `disk_stats.py` compare the
engine's own counters, which do not move with disk contention the way tok/s does.

## Do NOT use UPDATE.bat / START-HERE.bat --update here

They re-run setup, which rewrites run-config args. The manual steps below take ~10
minutes and leave every hand-tuned value intact.

## Lineage warning (read first)

This clone's git history does **not** share commits with the current upstream `main`
(the sak2k-bio fork force-reset its `main` to pure upstream). `git pull --ff-only` will
refuse. Update by creating a **new branch from `upstream/main`**, keeping the old
branch as the rollback point. Do **not** `git reset --hard` the old branch away.

## Update procedure (server code + engine — always together)

1. **Stop the server** (close the run bat; port 4444 must be free).
2. **Back up the docs** — `CUSTOM_3060_32gb.md` and `QUICK_UPDATE.md` are tracked on
   `3060_vram` but **absent upstream**, so the branch switch deletes them from disk:
   ```
   mkdir F:\STRATA\repo-docs-backup-<date>
   copy Strata\CUSTOM_3060_32gb.md Strata\QUICK_UPDATE.md F:\STRATA\repo-docs-backup-<date>\
   ```
3. **Fetch and branch**:
   ```
   git -C Strata fetch upstream --tags
   git -C Strata checkout -b update-<ver> v<ver>
   ```
   Branch from the **release tag**, not `upstream/main`: `main` can carry unreleased work
   between releases, and the engine you download is the release one. (0.2.2: `v0.2.2` is the
   tip of `upstream/main` as of this update, so the two agreed — check anyway.)
   The old branch (e.g. `update-0.1.45`) stays untouched as the rollback point.
4. **Restore the docs** from the backup into `Strata\` (they become untracked; commit
   them on the new branch if you want them tracked again).
5. **Engine** (not in git — a prebuilt download). **The packaging changed in 0.2.0: the engine
   is a `.tar.gz` now, not a zip, and it bundles the CUDA DLLs.**
   - Asset name: `strata-engine-<enginever>-windows-x86_64-cuda-cuda13.tar.gz`
      (**app 0.2.14 and 0.2.15 ship engine 0.1.60** — the app version and the engine version
      differ, and 0.2.15 rebuilds nothing for Windows x86; it is the 0.2.14 file).
      This PC: driver 616 → cuda13.
   - **Extract into an empty `engine\`.** The 0.1.60 package has 6 files and no longer carries
     `strata_kq.dll` (0.1.52 had 7 files including it), so `tar -xzf ... -C engine` over an
     existing folder leaves a stale DLL behind. Clear the folder first; the old engine is in
     `engine-backup-<oldver>\` either way.
   - `curl -L -C - -o F:\STRATA\strata-engine-<enginever>.tar.gz https://github.com/Niko1221/Strata/releases/download/v<APPVER>/strata-engine-<enginever>-windows-x86_64-cuda-cuda13.tar.gz`
   - Verify: `certutil -hashfile <file> SHA256` must equal the release asset digest
     (from the GitHub API `digest` field or the release page).
   - Backup: copy every file of `Strata\engine\` → `Strata\engine-backup-<oldver>\`.
   - Install: `tar -xzf <file> -C Strata\engine` — the archive is **flat** (no top folder):
     `strata.exe`, `strata-vision.exe`, `strata_kq.dll`, `BUILD.json`, `engine-component.json`,
     `cublas64_13.dll`, `cublasLt64_13.dll`. Check `list` first with `tar -tzf`.
   - Check `Strata\engine\BUILD.json` shows the new `version`.
   - The bundled cublas DLLs are found first (the exe's own folder); the venv's
     `lib_dirs` still supplies `cudart64_13.dll`, which the package does not carry.
6. **Python deps**: `git diff <oldbranch> <newtag> -- requirements.txt` — if
   changed: `.venv\Scripts\python.exe -m pip install -r requirements.txt`.
   (**0.2.7 added `truststore==0.10.5`, so this step is required, not optional.** 0.2.2 added
   nothing.)
7. **Config args**: check the new `src/program/generate.cpp` help text / DETAILS.md
   for removed flags. All 16 of ours survived 0.2.7 (`--pack`, `--native`, `--ple-gguf`,
   `--expert-profile`, `--expert-cache`, `--prefill`, `--spec`, `--mtp`, `--max-context`,
   `--kv`, `--kv-grow`, `--vram-reserve-mib`, `--resident-experts`, `--pcie-frac`,
   `--spec-min-p`, `--prompt-cache-every`).
   The 0.2 server also accepts `"exe": "component:strata-engine"`, but our absolute path
   keeps working and is what this install uses.
8. **Start** `run-coder-iq1_m.bat` and verify:
   - `curl http://127.0.0.1:4444/health` → `"loaded":true`
   - `GET /v1/status` (API key) → `"engine":"<new version>"`
   - One chat request returns a correct answer with sane `timings`.
   - Log tail: `session is up (engine <ver>)`, expert-cache slot count, no `WARNING`
     beyond the known resident-RAM fallback (expected on 32 GB).
   - On sm_86 the log names the new default switch table:
     `sm_86 (Ampere): 9 exact speed switches on by default ...` — that is the +10% decode win.

## Rollback (~2 minutes)

- Code: `git -C Strata checkout <old branch>` (e.g. `update-0.2.2`), then restore the docs
  from the matching `repo-docs-backup-<date>\` (the branch switch deletes them).
- Engine: copy `engine-backup-<oldver>\*` back over `engine\` (for 0.2.7: `engine-backup-0.1.47\`,
  which holds all 7 files).
- Config/bat: never in git — untouched by the update; keep manual backups if edited.
- Stop the server first: the engine binary is locked while it runs, and the copy fails
  with "used by another process". `taskkill /IM strata.exe /F` and `taskkill /IM python.exe /F`.

## Notes

- **Keep code and engine in sync.** They are separate artifacts and drift silently: the
  engine lives on disk in `engine\` while the server code comes from whichever git branch
  is checked out. A 0.1.43 engine under 0.1.40-era server code runs (the protocol is
  compatible) but you do not get the new server behaviour, and the branch's source will
  disagree with `engine\BUILD.json`. After any branch switch, confirm the two match.
- **Never commit conflict markers.** A `git stash pop` that collides with tracked copies
  writes `<<<<<<<` / `>>>>>>>` into the file; committing that poisons the docs for every
  later checkout, and `git checkout <branch>` will not heal it. Check with
  `findstr /c:"<<<<" <file>` before committing. Clean versions of both docs survive at
  commit `834ded74`.
- Timings live in the `timings` block of the chat response (`predicted_per_second` =
  decode, `prompt_per_second` = prefill) — not under `usage`.
- **Benchmarking an engine change on this box needs care.** The spread between two runs of
  the *same* engine (prefill 472–639 tok/s, decode 32.6–43.7) is as large as the difference
  between engines. So: swap the engine and restart, then compare **interleaved** rounds
  (A, B, A, B), warm up 6 requests before measuring, and hold free RAM constant at engine
  start — the resident expert pool is sized from MemAvailable, and a 15.6 GiB pool against a
  7.1 GiB one is not a fair comparison. `F:\STRATA\bench\ab_cycle.ps1` does one round
  (swap, restart, wait for the HTTP port, warm up, measure); `ab_warmed.ps1` is the
  measurement itself. The first run after a swap is always slow (the pool pages in).
- **Answers here are not byte-stable between runs.** Two identical greedy requests on the
  *same* engine gave different text on 2 of 3 prompts (the adaptive tier moves experts
  between VRAM and RAM). Never treat a text difference between two engines as a regression
  on this box; upstream's own identity gate is the reference.
- From 0.1.41 the CPU prefill share is on by default for our shape (CUDA, one GPU, no batch
  slots), but 0.1.42 added a memory guard: on this 32 GB box the engine declines it and logs
  `the CPU prefill share is not armed by default: ... only 4.7 GiB of RAM is free`. That is
  expected and correct here — do not force it on.
- Server and engine versions must match; a new server talks new protocol lines to the engine.
- Model files (Strata-data) are never touched by any step above.
- First start after an engine swap re-reads the model; expect the usual 1–3 min load and a slow
  first request while the resident expert pool pages in.
