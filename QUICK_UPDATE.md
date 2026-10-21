# QUICK_UPDATE.md — safely update Strata (RTX 3060 / 32 GB install)

Verified 2026-10-21: app 0.1.45 → 0.2.2 (engine 0.1.45 → 0.1.47), server + engine together,
custom config untouched (branch `update-0.2.2`; before that 2026-10-17, v0.1.43 → v0.1.45;
2026-10-12, v0.1.40.1 → v0.1.43; 2026-10-05, v0.1.38 → v0.1.39).
This install carries hand-tuned pieces: `strata-coder-iq1_m.json` + `run-coder-iq1_m.bat` (both
gitignored — git never touches them), calibrated `--pcie-frac 0.35` / `--spec-min-p 0.70`,
F:\STRATA paths, and port 4444.

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
     (app 0.2.2 ships engine 0.1.47; the app version and the engine version differ).
     This PC: driver 616 → cuda13.
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
6. **Python deps**: `git diff <oldbranch> upstream/main -- requirements.txt` — if
   changed: `.venv\Scripts\python.exe -m pip install -r requirements.txt`.
   (0.2.2: only `cmake`/`ninja` moved to `requirements-build.txt`; nothing new to install.)
7. **Config args**: check the new `src/program/generate.cpp` help text / DETAILS.md
   for removed flags. All of ours survived 0.2.2.
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

- Code: `git -C Strata checkout <old branch>` (e.g. `update-0.1.45`), then restore the docs
  from the matching `repo-docs-backup-<date>\` (the branch switch deletes them).
- Engine: copy `engine-backup-<oldver>\*` back over `engine\`.
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
