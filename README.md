# gdrive-eversolo-sync

Claude Code skill + standalone scripts for an **Eversolo DMP-A6** (or other
Zidoo-platform network music player). Two things in one repo:

1. **A Bash CLI to play music on the Eversolo from a Linux terminal** —
   transport (play/pause/next/seek/volume), now-playing, queue inspection,
   indexed library search across the SMB share, and M3U playlist generation
   with `smb://` URIs.
2. **An album-aware, one-way sync workflow** between a **Google Drive master
   library** and the Eversolo. It understands that the same album can live
   under different folder names on each side, and that file-format
   differences (FLAC ⇄ ALAC ⇄ AIFF ⇄ WAV) shouldn't trigger a sync. Anything
   that's only on the Eversolo or where Eversolo wins gets uploaded to a
   separate `Music-unsynced/` folder on Drive so the master stays clean.

> **Platform support:** developed and tested on **Linux** (Ubuntu 20.04).
> The control / library / sync scripts have **not been tested on macOS**;
> they rely on `gio` (GVFS) for SMB mounts and standard GNU userland (`bash`,
> `awk`, `find -printf`, `python3`). They may or may not work on macOS as-is.
> Patches welcome.

## What's here

```
SKILL.md                            Claude Code skill definition
scripts/
  eversolo                          device control + library search/playlist
  eversolo-make-plan                generate ~/sync-plan.md (album-level diff)
  eversolo-sync                     execute the plan (rclone copy; dry-run by default)
templates/
  Music-unsynced-README.md          uploaded to Drive Music-unsynced/ on first run
```

## Install

Either as a Claude Code skill (recommended for Claude users) or as
plain scripts (works without Claude).

### As a Claude Code skill

Clone into your skills dir:

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
ln -s "$PWD/scripts/eversolo"           ~/.local/bin/eversolo
ln -s "$PWD/scripts/eversolo-make-plan" ~/.local/bin/eversolo-make-plan
ln -s "$PWD/scripts/eversolo-sync"      ~/.local/bin/eversolo-sync
```

## One-time setup

### 1. Eversolo discovery

```bash
avahi-browse -rt _eversolo._tcp     # find your device
export EVERSOLO=192.168.x.y         # the device IP
```

The TXT record reveals the SMB share user/password (or shows the device's
defaults). Find your USB drive's volume UUID by mounting the share:

```bash
gio mount smb://$EVERSOLO/Share/      # then enter SMB credentials
ls /run/user/$UID/gvfs/smb-share*/    # one dir per volume; e.g. 7DEF-F569
export EVERSOLO_MUSIC_REL=7DEF-F569/Music
```

### 2. rclone

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

## Usage

```bash
# Discovery and control
eversolo status                           # current track + queue
eversolo play | pause | next | volume 150 # range is 0–200, NOT 0–100
eversolo index                            # index the SMB library locally
eversolo search bill evans                # AND-match across artist/album/track
eversolo playlist -o jazz.m3u8 audiophile

# Diff + plan + apply
eversolo-make-plan                        # writes ~/sync-plan.md
eversolo-make-plan --verify-tags          # add tag-comparison appendix (slower)
eversolo-sync                             # dry-run, all actions
eversolo-sync --apply                     # actually do it
eversolo-sync --apply --only PULL         # one phase at a time
```

After a sync, **manually trigger a rescan on the Eversolo** (Music app →
pull-down to refresh, or Settings → Music Library → Rescan). The HTTP API
can't drive the rescan — its scanner endpoints require an undiscovered
registration handshake the Eversolo phone app does.

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
| `REFACTOR`         | same album, different folder names — informational | (none) |
| `DUP-DRIVE`        | duplicate album folders on Drive (cleanup) | (none) |
| `DUP-EVERSOLO`     | duplicate album folders on Eversolo | (none) |
| `SKIP`             | matched on both sides | (none) |

Only `PULL` and `UPLOAD-UNSYNCED` get translated to `rclone copy` calls; the
others need human decisions and live in the plan as informational sections.

## Why the side channel

Drive is the **master**: rip → tag → upload to Drive. All other devices
(Eversolo, etc.) mirror Drive's `Music/` folder. If you let arbitrary content
flow back from a mirror to the master, you'll keep finding mis-named or
incomplete copies cluttering up the canonical library. So this skill sends
"interesting from Eversolo" to a separate `Music-unsynced/` directory at
Drive root — preserved as cloud backup, but never part of the official
library.

## Tested with

- **OS**: Ubuntu 20.04 (Linux). **Not tested on macOS** — the Bash control
  CLI and sync scripts depend on `gio` (GVFS), GNU `find -printf`, and the
  standard Linux userland. Some pieces may run on macOS unmodified, others
  (especially the GVFS SMB mount path) won't.
- Eversolo DMP-A6 firmware v1.5.75 (Android 11)
- rclone v1.74

## License

MIT.
