# Dropbox Deduplication Guide

Reference for removing duplicate photos and videos from Dropbox using CLI tools and GUI options.

---

## ⚠️ Before You Start

- **Enable Dropbox Version History** on your account before running any destructive commands — this is your safety net.
- **Always dry-run first.** Every tool below has a preview/list mode. Use it.
- Hash-based dedup finds *identical* files only. For "similar but not identical" photos (different edits, resizes, crops), see [Similar Photos](#similar-photos-not-identical) below.

---

## Option 1: rclone dedupe --by-hash (Recommended)

Best for: content duplicates across the entire Dropbox, even with different filenames or locations.

> **Note:** `rclone dedupe` without `--by-hash` targets duplicate *filenames*, which Dropbox doesn't allow natively — so `--by-hash` is required here to find content-identical files.

### Configure a Dropbox remote (first time only)

```bash
rclone config
# Choose "n" for new remote
# Name it: dropbox
# Select Dropbox from the list
# Follow the OAuth browser flow
```

### Dry run — list duplicates only

```bash
# Entire Dropbox
rclone dedupe --by-hash --dedupe-mode list dropbox:

# Specific folder
rclone dedupe --by-hash --dedupe-mode list dropbox:Photos
```

### Delete duplicates, keeping the newest copy

```bash
rclone dedupe --by-hash --dedupe-mode newest dropbox:Photos
```

### Dropbox API Rate Limiting

Dropbox enforces API rate limits and will block rclone mid-run with errors like:

```
NOTICE: Error too_many_requests/... Too many requests or write operations. Trying again in 300 seconds.
```

Always include `--tpslimit` flags to stay under the limit:

```bash
# Recommended: 4 TPS (safe for most accounts)
rclone dedupe --by-hash --dedupe-mode newest \
  --tpslimit 4 \
  --tpslimit-burst 4 \
  dropbox:

# Conservative: 2 TPS (if 4 still triggers rate limits)
rclone dedupe --by-hash --dedupe-mode newest \
  --tpslimit 2 \
  --tpslimit-burst 2 \
  dropbox:
```

- `--tpslimit` — max API transactions per second
- `--tpslimit-burst` — max burst above that limit (set equal to `--tpslimit` to prevent bursting)

The run will be slower but will complete without hitting the 300s retry wall.

### dedupe-mode options

| Mode          | Behaviour                                      |
|---------------|------------------------------------------------|
| `list`        | List duplicates only, no changes (dry run)     |
| `interactive` | Prompt for each duplicate group                |
| `newest`      | Keep most recently modified — best for photos  |
| `oldest`      | Keep oldest copy                               |
| `largest`     | Keep largest file — best for videos            |
| `smallest`    | Keep smallest file                             |
| `first`       | Keep first encountered, delete the rest        |
| `rename`      | Rename duplicates rather than delete           |

---

## Option 2: jdupes (Better for Cross-Folder Dedup)

Best for: same file present in multiple folders (e.g. `Camera Uploads/` and `Photos/2023/`). Works on a local copy of your Dropbox.

### Install

```bash
# macOS
brew install jdupes

# Ubuntu/Debian
sudo apt install jdupes
```

### Workflow

```bash
# 1. Pull Dropbox folder locally
rclone sync dropbox:Photos ~/dropbox-photos/

# 2. Preview duplicates (no changes)
jdupes -r ~/dropbox-photos/

# 3. Delete duplicates, keeping the first copy found per group
jdupes -r -d ~/dropbox-photos/

# 4. Push cleaned folder back
rclone sync ~/dropbox-photos/ dropbox:Photos
```

### Useful jdupes flags

| Flag | Meaning                                      |
|------|----------------------------------------------|
| `-r` | Recursive (scan subdirectories)              |
| `-d` | Delete duplicates (keeps first in each group)|
| `-N` | No-prompt mode (use with `-d` carefully)     |
| `-S` | Show size of duplicate sets                  |
| `-z` | Consider zero-byte files as duplicates       |

---

## Option 3: GUI Tool (Easiest)

**Easy Duplicate Finder** has a native Dropbox scan mode — it connects via the Dropbox API, scans file metadata, and lets you preview duplicates before deletion. Includes an undo feature.

Other GUI options:
- **dupeGuru** — cross-platform, good for photo libraries
- **digiKam** — powerful photo manager with built-in duplicate finder; can handle Dropbox via local sync

---

## Similar Photos (Not Identical)

Hash-based tools won't catch photos that are visually similar but differ in metadata, compression, or edits. For those:

- **digiKam** — fuzzy/perceptual image matching
- **dupeGuru (Image mode)** — configurable similarity threshold
- **Czkawka** — fast, open-source, supports similar image detection

Workflow: sync Dropbox locally → run tool → review matches manually → sync back with rclone.

---

## Quick Reference

```bash
# Audit duplicates (safe, no changes)
rclone dedupe --by-hash --dedupe-mode list dropbox:Photos

# Remove duplicates, keep newest (with rate limit protection)
rclone dedupe --by-hash --dedupe-mode newest --tpslimit 4 --tpslimit-burst 4 dropbox:Photos

# Full Dropbox scan (with rate limit protection)
rclone dedupe --by-hash --dedupe-mode newest --tpslimit 4 --tpslimit-burst 4 dropbox:

# Local dedup with jdupes
jdupes -r -d ~/dropbox-photos/
```

---

## Related Tools

- [rclone docs — dedupe](https://rclone.org/commands/rclone_dedupe/)
- [jdupes GitHub](https://github.com/jbruchon/jdupes)
- [Czkawka](https://github.com/qarmin/czkawka)
- [dupeGuru](https://dupeguru.voltaicideas.net/)
