# QUICK_UPDATE.md — safely update Strata (RTX 3060 / 32 GB install)

Verified 2026-10-17: v0.1.43 → v0.1.45, server + engine together, custom config untouched
(branch `update-0.1.45`; before that 2026-10-12, v0.1.40.1 → v0.1.43; 2026-10-05, v0.1.38 → v0.1.39).
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
   (0.1.45: unchanged.)
7. **Config args**: check the new `src/program/generate.cpp` help text / DETAILS.md
   for removed flags. All of ours survived 0.1.45.
8. **Start** `run-coder-iq1_m.bat` and verify:
   - `curl http://127.0.0.1:4444/health` → `"loaded":true`
   - `GET /v1/status` (API key) → `"engine":"<new version>"`
   - One chat request returns a correct answer with sane `timings`.
   - Log tail: `session is up (engine <ver>)`, expert-cache slot count, no `WARNING`
     beyond the known resident-RAM fallback (expected on 32 GB).

## Rollback (~2 minutes)

- Code: `git -C Strata checkout <old branch>` (e.g. `3060_vram`).
- Engine: copy `engine-backup-<oldver>\*` back over `engine\`.
- Config/bat: never in git — untouched by the update; keep manual backups if edited.

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
- From 0.1.41 the CPU prefill share is on by default for our shape (CUDA, one GPU, no batch
  slots), but 0.1.42 added a memory guard: on this 32 GB box the engine declines it and logs
  `the CPU prefill share is not armed by default: ... only 4.7 GiB of RAM is free`. That is
  expected and correct here — do not force it on.
- Server and engine versions must match; a new server talks new protocol lines to the engine.
- Model files (Strata-data) are never touched by any step above.
- First start after an engine swap re-reads the model; expect the usual 1–3 min load and a slow
  first request while the resident expert pool pages in.
