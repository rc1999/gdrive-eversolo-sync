# gdrive-eversolo-sync

Claude Code skill + standalone scripts for keeping an **Eversolo DMP-A6**
(or other Zidoo-platform network music player) mirrored from a **Google
Drive** master library.

The workflow is one-way (Drive → Eversolo) and album-aware: it understands
that the same album can live under different folder names on each side, and
that file-format differences (FLAC ⇄ ALAC ⇄ AIFF ⇄ WAV) shouldn't trigger a
sync. Anything that's only on the Eversolo or where Eversolo wins gets
uploaded to a separate `Music-unsynced/` folder on Drive so the master stays
clean.

> **Platform support:** developed and tested on **Linux** (Ubuntu 20.04).
> The scripts rely on rclone, GNU `awk`/`find`, and `python3` (with optional
> `mutagen` / `ffprobe` for tag verification). **Not tested on macOS** —
> may or may not work as-is.

## What's here

```
SKILL.md                            Claude Code skill definition
scripts/
  eversolo-make-plan                generate ~/sync-plan.md (album-level diff)
  eversolo-sync                     execute the plan (rclone copy; dry-run by default)
templates/
  Music-unsynced-README.md          uploaded to Drive Music-unsynced/ on first run
```

## Install

### As a Claude Code skill

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/rc1999/gdrive-eversolo-sync.git ~/.claude/skills/gdrive-eversolo-sync
```

Then in any Claude Code session, ask things like *"sync my Drive music to the
Eversolo"* or *"diff Drive Music against my Eversolo"* — Claude will pick up
the skill and follow the workflow.

### As standalone scripts

```bash
git clone https://github.com/rc1999/gdrive-eversolo-sync.git
cd gdrive-eversolo-sync
chmod +x scripts/*
ln -s "$PWD/scripts/eversolo-make-plan" ~/.local/bin/eversolo-make-plan
ln -s "$PWD/scripts/eversolo-sync"      ~/.local/bin/eversolo-sync
```

## One-time setup

### 1. rclone

Install rclone (any method — `apt`, `brew`, the official installer, or the
plain binary). Then configure two remotes:

```bash
rclone config            # n -> name=gmusic   -> drive    (scope=full, OAuth)
rclone config            # n -> name=eversolo -> smb      (host, user, pass)
```

If you encrypt the rclone config (`s` in the config menu), store the
password in `/tmp/rcpass`:

```bash
umask 077 && printf '%s' 'YOUR_CONFIG_PASS' > /tmp/rcpass
```

The skill auto-detects this file; otherwise pass `--password-command "cat /path"`.

### 2. Find your Eversolo's USB volume UUID

The Eversolo exposes its attached storage by volume UUID under the SMB
share. Find it once and bake it into your `EVERSOLO_BASE`:

```bash
rclone --password-command "cat /tmp/rcpass" lsd eversolo:Share
# -> 7DEF-F569      <-- your USB drive's UUID
# -> Storage
```

So your music root is `eversolo:Share/7DEF-F569/Music` (substitute your UUID).

## Usage

```bash
eversolo-make-plan                        # writes ~/sync-plan.md
eversolo-make-plan --verify-tags          # add tag-comparison appendix (slower)
eversolo-sync                             # dry-run, all actions
eversolo-sync --apply                     # actually do it (additive only)
eversolo-sync --apply --force             # also process FORMAT-REPLACE (destructive)
eversolo-sync --apply --only PULL         # one phase at a time
eversolo-sync --apply --only UPLOAD-UNSYNCED
```

`--force` enables `FORMAT-REPLACE` — albums where the same content exists on
both sides but the file format differs in the canonical folders. With
`--force`, those are processed via `rclone sync` (rather than `copy`), which
**deletes files on the Eversolo that aren't on Drive** (e.g. an old AIFF
copy of an album you've since re-ripped to FLAC) and copies in Drive's
versions. Always run a dry-run (`eversolo-sync --force`, no `--apply`) and
inspect the diff first.

After a sync, **manually trigger a rescan on the Eversolo** (Music app →
pull-down to refresh, or Settings → Music Library → Rescan). The Eversolo's
HTTP API exposes scanner endpoints, but they require an undiscovered
registration handshake the phone app does — they can't be driven from here.

## How it decides what to sync

Album matching uses two passes (both case + diacritics + punctuation
insensitive):

1. **Pass 1**: normalized `(artist | album)` exact match.
2. **Pass 2** (fallback): album-name only with `[brackets]`, `(parens)`, and
   catalog-code prefixes (e.g. `AS09 - `, `[CS 8271]`, `[mono]`, `[Disc N]`)
   stripped, and `Various` / `Various Artists` / `Compilations` /
   `Soundtracks` collapsed into one bucket.

**File-format differences alone never trigger a sync.** The Eversolo can play
ALAC just fine; if Drive has FLAC and Eversolo has ALAC of the same album, no
copy happens.

The plan emits these action classes:

| Action | Meaning | Sync direction |
|---|---|---|
| `PULL`             | on Drive only — copy down  | Drive → Eversolo |
| `UPLOAD-UNSYNCED`  | on Eversolo only or Eversolo wins | Eversolo → Drive `Music-unsynced/` |
| `FORMAT-REPLACE`   | same album, format differs in canonical folders — opt-in via `--force` | Drive → Eversolo (`rclone sync`, destructive) |
| `REFACTOR`         | same album, different folder names — informational | (none) |
| `DUP-DRIVE`        | duplicate album folders on Drive (cleanup) | (none) |
| `DUP-EVERSOLO`     | duplicate album folders on Eversolo | (none) |
| `SKIP`             | matched on both sides | (none) |

`PULL` and `UPLOAD-UNSYNCED` translate to additive `rclone copy` calls.
`FORMAT-REPLACE` translates to `rclone sync` (destructive at the album-folder
level) and only runs with `--force`. The others need human decisions and
live in the plan as informational sections.

## Why the side channel

Drive is the **master**: rip → tag → upload to Drive. All other devices
(Eversolo, etc.) mirror Drive's `Music/` folder. If you let arbitrary content
flow back from a mirror to the master, you'll keep finding mis-named or
incomplete copies cluttering up the canonical library. So this skill sends
"interesting from Eversolo" to a separate `Music-unsynced/` directory at
Drive root — preserved as cloud backup, but never part of the official
library.

## Tag verification (optional)

`eversolo-make-plan --verify-tags` reads embedded `artist` / `albumartist` /
`album` / `date` tags from one representative track per priority album. For
Drive, it fetches partial bytes via `rclone cat --head 500000` (and `--tail`
for M4A — the `moov` atom can live at the end of the file). It uses
`mutagen` first and falls back to `ffprobe` for AIFF variants mutagen
doesn't grok. The result is appended to the plan as a Tag-verification
appendix, useful for catching real false-matches the path-based normalizer
can't see (e.g. wrong-artist tags on Drive).

## Tested with

- **OS**: Ubuntu 20.04 (Linux). **Not tested on macOS.**
- Eversolo DMP-A6 firmware v1.5.75 (Android 11)
- rclone v1.74

## License

MIT.
