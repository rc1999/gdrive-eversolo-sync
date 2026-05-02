---
name: gdrive-eversolo-sync
description: Mirror a Google Drive music library (master) to an Eversolo network music player (DMP-A6 and similar Zidoo-based devices), one-way and album-aware. Use when the user wants to discover an Eversolo on the LAN, browse its SMB library, control playback, generate a content-based diff between Drive and the Eversolo, and run the sync. Format-only differences (FLAC ⇄ ALAC ⇄ AIFF) never trigger a sync. Eversolo-only or Eversolo-wins material is uploaded to a separate `Music-unsynced/` folder on Drive so the master stays clean.
---

# Drive ↔ Eversolo sync

End-to-end workflow for keeping an Eversolo DMP-A6 (or other Zidoo-platform
network music player) mirrored from a Google Drive master library, with safe
album-level diffing and a side channel for material that shouldn't pollute the
master.

## When to invoke this skill

Trigger on requests like:
- "find/discover music devices on my network" → discovery + control
- "control / play / pause / queue on my Eversolo" → use `scripts/eversolo`
- "diff my Eversolo library against Google Drive"
- "sync my music from Drive to Eversolo"
- "rclone won't let me push only what's missing on Eversolo"
- "build a playlist of <X> from the library"
- Any mention of Eversolo + Google Drive together

## What's in this skill

```
scripts/
  eversolo            # device control + library search/playlist (no rclone needed)
  eversolo-make-plan  # generate ~/sync-plan.md (Drive ⇄ Eversolo album-level diff)
  eversolo-sync       # execute the plan (rclone copy; dry-run by default)
templates/
  Music-unsynced-README.md   # uploaded to user's Drive Music-unsynced/ on first run
```

All three scripts read configuration from environment variables and have
sensible defaults. Run any of them with `--help` to see options.

## Workflow

### 1. Discovery and control (no rclone, no Drive needed)

```bash
# Find Eversolo / AirPlay / Spotify Connect / Tidal Connect on the LAN
avahi-browse -rt _eversolo._tcp
avahi-browse -rt _raop._tcp

# Use the device control script (set EVERSOLO=<ip> if discovery didn't auto-fill)
scripts/eversolo status         # now-playing + volume + queue
scripts/eversolo play|pause|next|prev|seek <ms>
scripts/eversolo volume 150     # range is 0–200, NOT 0–100
scripts/eversolo queue 50

# Library access via SMB share `Share/<volume-uuid>/Music/`
# Default user is set in device Settings → Storage → Network Share
# Mount with gio:
gio mount smb://<eversolo-ip>/Share/

# Build a local index of the library, then search and make playlists
scripts/eversolo index                            # ~1–2 min over Wi-Fi
scripts/eversolo search bill evans
scripts/eversolo playlist -o album.m3u8 audiophile
```

The control script talks to the Eversolo's HTTP API on port 9529
(unauthenticated on the LAN). The Zidoo-style endpoints used:

- `/ZidooControlCenter/getModel`            — device info
- `/ZidooMusicControl/v2/getState`          — now-playing
- `/ZidooMusicControl/v2/getVolume`         — read volume (max=200)
- `/ZidooMusicControl/v2/setVolume?volume=<0-200|up|down|mute>`
- `/ZidooMusicControl/v2/playOrPause`       — toggle (single endpoint)
- `/ZidooMusicControl/v2/playNext` / `playLast`
- `/ZidooMusicControl/v2/seekTo?time=<ms>`
- `/ZidooMusicControl/v2/getPlayQueue?start=&count=`

Library-browse endpoints (`/ZidooMusicControl/v3/getAlbumList`,
`/MusicService/v2/...`) exist but require an undiscovered registration
handshake — do NOT try to brute-force parameters. Use SMB for library reads.

### 2. First-time rclone setup (one-off)

```bash
# Install rclone if needed
curl -fsSL https://rclone.org/install.sh | sudo bash
# or user-only:
curl -fsSL -o /tmp/rclone.zip https://downloads.rclone.org/rclone-current-linux-amd64.zip \
  && unzip -q /tmp/rclone.zip -d /tmp \
  && install -m 0755 /tmp/rclone-v*-linux-amd64/rclone ~/.local/bin/rclone

# Configure two remotes (interactive; OAuth needs a browser):
rclone config           # -> n -> name=gmusic    -> drive  (scope=full)
rclone config           # -> n -> name=eversolo  -> smb    (host, user, pass)

# If config is password-encrypted, store the password in /tmp/rcpass mode 0600
# (then pass --password-command "cat /tmp/rcpass" to every rclone call).
umask 077 && printf '%s' 'YOUR_PASS' > /tmp/rcpass
```

Default remote names this skill expects:
- `gmusic:Music/`           — Drive **master** library
- `gmusic:Music-unsynced/`  — Drive holding pen for Eversolo-only/wins
- `eversolo:Share/<UUID>/Music/`  — Eversolo SMB

Override via env: `DRIVE_REMOTE`, `DRIVE_MUSIC_PATH`, `DRIVE_UNSYNCED_PATH`,
`EVERSOLO_REMOTE`, `EVERSOLO_BASE`.

### 3. Generate the diff plan

```bash
scripts/eversolo-make-plan -o ~/sync-plan.md
```

What it does:
1. Lists every audio file on each side (rclone `lsf -R --files-only` filtered
   to FLAC/MP3/M4A/WAV/AIF/AIFF/DSF/DFF/OGG/ALAC).
2. Two-pass album matching:
   - **Pass 1**: normalized `(artist | album)` — case + diacritics + punctuation
     insensitive.
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

When the user wants higher-confidence diffs, run with `--verify-tags` to
sample one track per album (head 500 KB; tail for M4A) and confirm artist /
album / albumartist embedded tags. The tag pass surfaces real false-matches
the path-based normalizer can't see (e.g. wrong-artist tags on Drive).

### 4. Apply the sync

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
- **The Eversolo volume range is 0–200, not 0–100.** 200 = 0 dB, 100 ≈ -30 dB.

## Common pitfalls

- **rclone `cat --head 200K` does not work** (rclone's `--head` takes raw
  bytes). Use `--head 200000`. Same for `--tail`.
- **`gio mount` with empty/anonymous credentials returns a friendly listing
  for shares you can't actually read.** Always provide username + password.
- **The Eversolo's HTTP API on :9529 has no auth on the LAN.** That's by
  design but worth noting — the device's mDNS TXT broadcasts `password=`
  (often empty). Encourage the user to set a non-empty admin password if
  guests share the Wi-Fi.
- **`MusicScanner/*` and `MediaScanner/*` endpoints return 801** ("application
  element is not registered"). The Eversolo phone app does a registration
  handshake we haven't reverse-engineered. To trigger a library rescan, the
  user has to use the device UI (Music app → pull-down to refresh, or
  Settings → Music Library → Rescan).
- **mutagen fails on some `.aif` variants** with `unsupported format`. Fall
  back to `ffprobe -show_format -show_streams` for those.

## After a sync, prompt the user to

1. Open the Eversolo Music app and pull-down-to-refresh (or Settings →
   Music Library → Rescan) — the API can't trigger this remotely.
2. Verify a sample of the new artist folders shows up (use the PULL list
   from `~/sync-plan.md` as a check-list).

## Tag-verification appendix (when `--verify-tags` is used)

The plan generator's tag pass:
- Reads ~one track per album from each side.
- For Drive, fetches partial bytes via `rclone cat --head 500000`; if mutagen
  fails on M4A, retries with `--tail 500000` (M4A `moov` atom can live at the
  end of the file).
- Compares `artist` / `albumartist` / `album` / `date` tags after the same
  `n_basic` normalization used for paths.
- Emits a comparison block that's appended to the plan as the final section.

The tag pass is OPTIONAL; default plans rely on path-based matching alone.
Use `--verify-tags` when the path-only result has many ambiguous Pass-2
matches.
