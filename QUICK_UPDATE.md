# QUICK_UPDATE.md — safely update Strata (RTX 3060 / 32 GB install)

<<<<<<< Updated upstream
Verified 2026-10-05: v0.1.38 → v0.1.39, server + engine together, custom config untouched.
This install carries hand-tuned pieces: `strata-coder-iq1_m.json` + `run-coder-iq1_m.bat` (both
gitignored — git never touches them), calibrated `--pcie-frac 0.35` / `--spec-min-p 0.70`,
F:\STRATA paths, port 4444, and a local `setup.py` disk-check patch.

## Do NOT use UPDATE.bat / START-HERE.bat --update here

They re-run setup, which rewrites run-config args. The manual steps below take ~5 minutes and
leave every hand-tuned value intact.

## Update procedure (server code + engine — always together)

1. **Stop the server** (close the run bat; port 4444 must be free).
2. **Stash local changes**: `git -C Strata status` — stash every modified file,
   e.g. `git -C Strata stash push -m "local patches" -- setup.py`.
3. **Pull**: `git -C Strata pull --ff-only`. Fails if anything is dirty — stash first, never force.
4. **Restore**: `git -C Strata stash pop` (should auto-merge; resolve by hand if not).
5. **Engine** (not in git — a prebuilt download):
   - Release notes pick the zip: `strata-windows-x64.zip` (CUDA 13, driver 580+) or
     `...-cuda12.zip` (driver 528+). This PC: driver 616 → cuda13.
   - `curl -L -o %TEMP%\strata-engine.zip https://github.com/Niko1221/Strata/releases/download/v<VER>/strata-windows-x64.zip`
   - Verify: `certutil -hashfile %TEMP%\strata-engine.zip SHA256` must equal the release asset digest.
   - Backup: copy `strata.exe`, `strata-vision.exe`, `BUILD.json` → `Strata\engine-backup-<oldver>\`.
   - Install: PowerShell `Expand-Archive -Path $env:TEMP\strata-engine.zip -DestinationPath Strata\engine -Force`.
6. **Python deps**: `git diff <oldsha>..<newsha> -- requirements.txt` — if changed:
   `.venv\Scripts\python.exe -m pip install -r requirements.txt`. (0.1.39: unchanged.)
7. **Config args**: check the release notes / new `src/program/generate.cpp` for removed flags.
   All of ours survived 0.1.39.
8. **Start** `run-coder-iq1_m.bat` and verify:
   - `curl http://127.0.0.1:4444/health` → `"loaded":true`
   - `GET /v1/status` (with API key) → `"engine":"<new version>"`
   - One chat request; then `strata-coder-iq1_m.log` tail looks like a normal start.

## Rollback

- Engine: copy `engine-backup-<ver>\*` back over `engine\`.
- Code: `git -C Strata reset --hard <old sha>`; local patches re-apply from the stash entry.
- Config/bat: not in git — keep your own backup copy if you edit them.

## Notes

- Server and engine versions must match; a new server talks new protocol lines to the engine.
- Model files (Strata-data) are never touched by any step above.
- First start after an engine swap re-reads the model; expect the usual 1–3 min load and a slow
  first request while the resident expert pool pages in.
=======
Verified 2026-10-07: v0.1.39 → v0.1.40.1 (code + engine together), config untouched.
Hand-tuned pieces that every update must preserve: `strata-coder-iq1_m.json` +
`run-coder-iq1_m.bat` (both gitignored — git never touches them), calibrated
`--pcie-frac 0.35` / `--spec-min-p 0.70`, F:\STRATA paths, port 4444.

## Lineage warning (read first)

This clone's git history does **not** share commits with the current upstream `main`
(the sak2k-bio fork force-reset its `main` to pure upstream). `git pull --ff-only` will
refuse. The update is: create a **new branch from `upstream/main`**, keep the old
branch as the rollback point. Do **not** `git reset --hard` the old branch away.

## Do NOT use UPDATE.bat / START-HERE.bat --update here

They re-run setup, which rewrites run-config args. The manual steps below take ~10
minutes and leave every hand-tuned value intact.

## Update procedure (code + engine — always together)

1. **Stop the server** (close the run bat; port 4444 must be free).
2. **Back up tracked docs** — files committed in the old lineage but absent upstream
   are deleted from disk by the branch switch:
   ```
   mkdir F:\STRATA\repo-docs-backup-<date>
   copy Strata\CUSTOM_3060_32gb.md Strata\QUICK_UPDATE.md F:\STRATA\repo-docs-backup-<date>\
   ```
3. **Fetch and branch**:
   ```
   git -C Strata fetch upstream
   git -C Strata checkout -b update-<ver> upstream/main
   ```
   The old branch (e.g. `3060_vram`) stays untouched as the rollback point.
4. **Restore the docs** from the backup into `Strata\` (they become untracked; commit
   them on the new branch if you want them tracked again).
5. **Engine** (not in git — a prebuilt download):
   - `strata-windows-x64.zip` (CUDA 13, driver 580+) or `...-cuda12.zip` (528+).
     This PC: driver 616 → cuda13.
   - `curl -L -o F:\STRATA\strata-engine-<ver>.zip https://github.com/Niko1221/Strata/releases/download/v<VER>/strata-windows-x64.zip`
   - Verify: `certutil -hashfile <zip> SHA256` must equal the release asset digest
     (from the GitHub API `digest` field or the release page).
   - Backup: copy `strata.exe`, `strata-vision.exe`, `BUILD.json` →
     `Strata\engine-backup-<oldver>\`.
   - Install: PowerShell `Expand-Archive -Path <zip> -DestinationPath Strata\engine -Force`.
   - Check `Strata\engine\BUILD.json` shows the new `version`.
6. **Python deps**: `git diff <oldbranch> upstream/main -- requirements.txt` — if
   changed: `.venv\Scripts\python.exe -m pip install -r requirements.txt`.
   (0.1.40: unchanged.)
7. **Config args**: check the new `src/program/generate.cpp` help text / DETAILS.md
   for removed flags. All of ours survived 0.1.40 (`--kv-grow` is new, opt-in).
8. **Start** `run-coder-iq1_m.bat` and verify:
   - `curl http://127.0.0.1:4444/health` → `"loaded":true`
   - `GET /v1/status` (API key) → `"engine":"<new version>"`
   - One chat request returns a correct answer with sane `timings`.
   - Log tail: `session is up (engine <ver>)`, expert-cache slot count, the chosen
     `prompt chunk auto: N tokens` line, no `WARNING` beyond the known resident-RAM
     fallback (expected on 32 GB).

## Rollback (~2 minutes)

- Code: `git -C Strata checkout <old branch>` (e.g. `3060_vram`).
- Engine: copy `engine-backup-<oldver>\*` back over `engine\`.
- Config/bat: never in git — untouched by the update; keep manual backups if edited.

## Notes

- Server and engine versions must match; setup enforces a minimum engine version.
- Model files (Strata-data) are never touched by any step above.
- First start after an engine swap re-reads the model: expect the usual 1–3 min load
  and a slow first request while the resident expert pool pages in.
- Engine zips accumulate at `F:\STRATA\strata-engine-*.zip` (~126 MB each); delete old
  ones once the update is verified.
>>>>>>> Stashed changes
