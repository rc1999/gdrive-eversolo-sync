---
name: gdrive-eversolo-sync
description: One-way, album-aware sync from a Google Drive master music library to an Eversolo network music player (DMP-A6 and other Zidoo-platform devices). Use when the user wants to diff Drive against the Eversolo, generate a content-based sync plan, and run the sync. Format-only differences (FLAC ⇄ ALAC ⇄ AIFF ⇄ WAV) never trigger a sync. Eversolo-only or Eversolo-wins material is uploaded to a separate Music-unsynced/ folder on Drive so the master stays clean. Tested on Linux (Ubuntu 20.04); not tested on macOS.
---

# Drive → Eversolo sync

End-to-end workflow for keeping an Eversolo DMP-A6 (or other Zidoo-platform
network music player) mirrored from a Google Drive master library, with safe
album-level diffing and a side channel for material that shouldn't pollute the
master.

**Platform note**: developed and tested on Linux only (Ubuntu 20.04). The
scripts use `rclone`, GNU userland, and `python3` (with optional `mutagen` /
`ffprobe` for tag verification). **Not tested on macOS** — assume things may
break there unless the user says otherwise.

## When to invoke this skill

Trigger on requests like:
- "diff my Eversolo library against Google Drive"
- "sync my music from Drive to Eversolo"
- "rclone won't let me push only what's missing on Eversolo"
- "what new albums will get pulled to my music player?"
- Any mention of Eversolo + Google Drive together involving content sync

This skill does **not** handle Eversolo playback control or library browsing.
The Eversolo's HTTP API exposes a transport endpoint set on port 9529, but
the actually-useful parts (play a specific track / album, trigger library
rescan, browse the library) are all gated behind a registration handshake
the Eversolo phone app performs — those endpoints accept calls and return
`status:200` but silently no-op without the right credentials. Use the
Eversolo phone app or the device touchscreen for playback. This skill stays
focused on the one thing rclone can reliably automate: file sync.

## What's in this skill

```
scripts/
  eversolo-make-plan  # generate ~/sync-plan.md (Drive ⇄ Eversolo album-level diff)
  eversolo-sync       # execute the plan (rclone copy; dry-run by default)
templates/
  Music-unsynced-README.md   # uploaded to user's Drive Music-unsynced/ on first run
```

Both scripts read configuration from environment variables and have sensible
defaults. Run with `--help` to see options.

## Workflow

### 1. Prereqs (one-off)

```bash
# rclone installed; two remotes configured:
#   gmusic   (drive)   — your Google Drive
#   eversolo (smb)     — your Eversolo's SMB share
rclone config
# If config is password-encrypted, store the password in /tmp/rcpass mode 0600
# (the scripts auto-detect this file):
umask 077 && printf '%s' 'YOUR_PASS' > /tmp/rcpass
```

Default remote names this skill expects:
- `gmusic:Music/`           — Drive **master** library
- `gmusic:Music-unsynced/`  — Drive holding pen for Eversolo-only/wins
- `eversolo:Share/<UUID>/Music/`  — Eversolo SMB (replace `<UUID>` with the
  USB drive's volume UUID; see `rclone lsd eversolo:Share`)

Override via env: `DRIVE_MASTER`, `DRIVE_UNSYNCED`, `EVERSOLO_BASE`,
`RCLONE_PASS_FILE`, `EVERSOLO_PLAN`.

### 2. Generate the diff plan

```bash
scripts/eversolo-make-plan -o ~/sync-plan.md
# optional: tag-comparison pass on priority items (slower)
scripts/eversolo-make-plan --verify-tags
```

What it does:
1. Lists every audio file on each side via `rclone lsf -R --files-only`,
   filtered to FLAC/MP3/M4A/WAV/AIF/AIFF/DSF/DFF/OGG/ALAC.
2. Two-pass album matching:
   - **Pass 1**: normalized `(artist | album)` — case + diacritics +
     punctuation insensitive.
   - **Pass 2** (fallback): album-name only with `[brackets]`, `(parens)`,
     and catalog-code prefixes (e.g. `AS09 -`, `[CS 8271]`, `[mono]`,
     `[Disc N]`) stripped, and `Various`/`Various Artists`/`Compilations`/
     `Soundtracks` collapsed into one bucket.
3. **Format-only differences never trigger a sync.**
4. Writes a Markdown report with a machine-readable section at the end:
   ```
   PULL              drive    <path>
   UPLOAD-UNSYNCED   eversolo <path>   reason:...
   REFACTOR          eversolo <path>   canonical-drive:<path>
   DUP-DRIVE         drive    <path>
   DUP-EVERSOLO      eversolo <path>
   SKIP              eversolo <path>
   ```

### 3. Apply the sync

```bash
scripts/eversolo-sync                              # dry-run, all actions
scripts/eversolo-sync --apply                      # actually do it
scripts/eversolo-sync --apply --only PULL          # one phase at a time
scripts/eversolo-sync --apply --only UPLOAD-UNSYNCED
```

The script reads `~/sync-plan.md`'s machine-readable section and runs
`rclone copy` (additive — never deletes) for each `PULL` and `UPLOAD-UNSYNCED`
line. Other actions (`REFACTOR`, `DUP-*`) are informational and produce no
rclone commands; they need human decisions.

### 4. After the sync — rescan on the Eversolo (manual)

Tell the user to trigger a rescan via the Eversolo Music app
(pull-down-to-refresh on the album/artist list) or
**Settings → Music Library → Rescan / Refresh**. The HTTP API can't drive
this — its `MusicScanner/*` and `MediaScanner/*` endpoints return
`status:801` ("application element is not registered") without the phone
app's handshake.

## Critical conventions

- **Sync direction is one-way: Drive → Eversolo.** Never auto-sync from
  `Music-unsynced/` to a mirror device.
- **Drive is the master.** If something is on Eversolo and not on Drive, it
  goes to `Music-unsynced/` on Drive — not into the master.
- **Format-only differences (FLAC ⇄ ALAC ⇄ AIFF ⇄ WAV) are never a sync
  trigger.**
- **Don't auto-dedupe.** Drive often has near-duplicate album folders
  (case / punctuation / `[Disc N]` variants). The plan flags them under
  `DUP-DRIVE` for the user to review — do not delete without confirmation;
  some `[Disc N]` "duplicates" are legitimate distinct content.
- **One canonical PULL path per album key.** When Drive has multiple folders
  for the same normalized album, only emit ONE `PULL` line so duplicates
  don't propagate to the Eversolo.

## Common pitfalls

- **rclone `cat --head 200K` does not work** (rclone's `--head` takes raw
  bytes). Use `--head 200000`. Same for `--tail`.
- **`MusicScanner/*` and `MediaScanner/*` endpoints return 801** ("application
  element is not registered"). The Eversolo phone app does a registration
  handshake we haven't reverse-engineered. To trigger a library rescan, the
  user has to use the device UI.
- **`playMusic`, `openFile`, etc. for triggering playback by path also
  silently no-op** — they accept the call and return `status:200` but don't
  actually play anything. Same gating as the scanner. Don't try to drive
  Eversolo playback from this skill.
- **mutagen fails on some `.aif` variants** with `unsupported format`. Fall
  back to `ffprobe -show_format -show_streams` for those.

## Tag-verification appendix (when `--verify-tags` is used)

The plan generator's optional tag pass:
- Reads ~one track per album from each side.
- For Drive, fetches partial bytes via `rclone cat --head 500000`; if mutagen
  fails on M4A, retries with `--tail 500000` (M4A `moov` atom can live at the
  end of the file).
- Compares `artist` / `albumartist` / `album` / `date` tags after the same
  normalization used for paths.
- Emits a comparison block that's appended to the plan as the final section.

The tag pass is OPTIONAL; default plans rely on path-based matching alone.
Use `--verify-tags` when the path-only result has many ambiguous Pass-2
matches.
