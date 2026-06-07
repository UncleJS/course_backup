# Module 04 — `dump` / `xfsdump` — Filesystem-Level Backup
[![CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey)](./LICENSE.md)
[![RHEL 10](https://img.shields.io/badge/platform-RHEL%2010-red)](https://access.redhat.com/products/red-hat-enterprise-linux)
[![Backup](https://img.shields.io/badge/Backup-RHEL%2010-blue)](https://access.redhat.com/products/red-hat-enterprise-linux)

## Learning Objectives

By the end of this module you will be able to:

- Understand the difference between file-level and filesystem-level backup
- Perform level-0 through level-9 dumps with `dump` (ext4) and `xfsdump` (XFS)
- Restore individual files and entire filesystems with `restore` and `xfsrestore`
- Understand why XFS is the default filesystem in RHEL 10 and what that means for tools
- Schedule dump-based backups as part of a retention strategy
- Perform bare-metal recovery using `xfsdump`

---

## Table of Contents

- [1. Filesystem-Level vs File-Level Backup](#1-filesystem-level-vs-file-level-backup)
- [2. RHEL 10 Default Filesystem: XFS](#2-rhel-10-default-filesystem-xfs)
- [3. Part A: `dump` and `restore` (for ext4/ext2/ext3)](#3-part-a-dump-and-restore-for-ext4ext2ext3)
  - [3.1 Installing dump](#31-installing-dump)
  - [3.2 The dump level system](#32-the-dump-level-system)
  - [3.3 Level-0 (full) dump](#33-level-0-full-dump)
  - [3.4 The `/etc/dumpdates` file](#34-the-etcdumpdates-file)
  - [3.5 Incremental dump (level 1+)](#35-incremental-dump-level-1)
  - [3.6 Dump options reference](#36-dump-options-reference)
  - [3.7 Restoring with `restore`](#37-restoring-with-restore)
- [4. Part B: `xfsdump` and `xfsrestore` (for XFS — RHEL 10 default)](#4-part-b-xfsdump-and-xfsrestore-for-xfs--rhel-10-default)
  - [4.1 Installing xfsdump](#41-installing-xfsdump)
  - [4.2 xfsdump concepts](#42-xfsdump-concepts)
  - [4.3 Level-0 (full) dump of an XFS filesystem](#43-level-0-full-dump-of-an-xfs-filesystem)
  - [4.4 xfsdump options reference](#44-xfsdump-options-reference)
  - [4.5 Incremental dumps](#45-incremental-dumps)
  - [4.6 Viewing the xfsdump inventory](#46-viewing-the-xfsdump-inventory)
  - [4.7 Estimating dump size](#47-estimating-dump-size)
  - [4.8 Dumping over SSH (piped to remote)](#48-dumping-over-ssh-piped-to-remote)
- [5. Restoring with `xfsrestore`](#5-restoring-with-xfsrestore)
  - [5.1 Interactive restore (recommended)](#51-interactive-restore-recommended)
  - [5.2 Restore entire filesystem](#52-restore-entire-filesystem)
  - [5.3 Applying incremental restores in order](#53-applying-incremental-restores-in-order)
  - [5.4 Restore a subtree (directory)](#54-restore-a-subtree-directory)
  - [5.5 Restore a single file](#55-restore-a-single-file)
  - [5.6 xfsrestore options reference](#56-xfsrestore-options-reference)
- [6. XFS Freeze for Consistent Live Backups](#6-xfs-freeze-for-consistent-live-backups)
- [7. Comparing dump/xfsdump Approaches](#7-comparing-dumpxfsdump-approaches)
  - [When to use `xfsdump`](#when-to-use-xfsdump)
  - [When to use `tar` or `rsync` instead](#when-to-use-tar-or-rsync-instead)
  - [Tool decision matrix](#tool-decision-matrix)
- [8. Automation Script](#8-automation-script)
- [Lab Exercises](#lab-exercises)
  - [Lab 04-1: Identify your filesystem types](#lab-04-1-identify-your-filesystem-types)
  - [Lab 04-2: Full xfsdump of /boot or /home](#lab-04-2-full-xfsdump-of-boot-or-home)
  - [Lab 04-3: Make changes and run incremental](#lab-04-3-make-changes-and-run-incremental)
  - [Lab 04-4: Interactive restore](#lab-04-4-interactive-restore)
  - [Lab 04-5: Run the automated backup script](#lab-04-5-run-the-automated-backup-script)
- [Review Questions](#review-questions)
- [Answers to Review Questions](#answers-to-review-questions)

---

## 1. Filesystem-Level vs File-Level Backup

File-level tools (tar, rsync) operate on the VFS layer — they open files through the kernel and copy them one by one. Filesystem-level tools operate at a lower level — they read the filesystem structure directly.

| Aspect | File-level (tar/rsync) | Filesystem-level (dump/xfsdump) |
|--------|----------------------|--------------------------------|
| What is backed up | Individual files and dirs | Raw filesystem data structures |
| Metadata preserved | Yes (with flags) | Yes — at the FS level |
| Speed on large FS | Slower (open each file) | Faster (sequential FS read) |
| Works across filesystems | Yes | No — FS-specific tools |
| Live backup (mounted FS) | Possible (with risk) | Supported (with limitations) |
| Incremental | Via snapshot file | Native level 0-9 system |
| Restore granularity | File/dir | File/dir or full FS |

[↑ Table of Contents](#table-of-contents)

---

## 2. RHEL 10 Default Filesystem: XFS

RHEL 10 uses **XFS** as the default filesystem for all partitions (root, `/home`, `/var`, etc.). This is important because:

- `dump`/`restore` are designed for **ext2/ext3/ext4** filesystems — they do not work on XFS
- XFS has its own dedicated tools: **`xfsdump`** and **`xfsrestore`**
- On a default RHEL 10 install, **every partition (including `/boot`) is XFS** — ext4 appears only on manually customised or migrated systems

```bash
# Verify your filesystem types
df -T
lsblk -f

# Confirm /boot is XFS (RHEL 10 default)
df -T /boot
```

[↑ Table of Contents](#table-of-contents)

---

## 3. Part A: `dump` and `restore` (for ext4/ext2/ext3)

### 3.1 Installing dump

> **⚠️ RHEL 10 availability:** the classic `dump`/`restore` package is **not available in RHEL 10 repositories** (BaseOS, AppStream, or EPEL 10) — `dnf install dump` fails on a stock system. Treat Part A as **reference material** for older or non-RHEL systems that still use ext4. On RHEL 10 the practical tool is `xfsdump` (Part B).

```bash
# On systems where the package exists (older distributions with ext4):
sudo dnf install -y dump
```

### 3.2 The dump level system

`dump` uses a **level number (0–9)** to define the scope of the backup:

| Level | What it backs up |
|-------|-----------------|
| 0 | Everything — full dump |
| 1 | All files changed since the last level-0 dump |
| 2 | All files changed since the last level-1 dump |
| ... | ... |
| 9 | All files changed since the last level-8 dump |

This enables flexible backup schedules. A common scheme:
- Level 0 monthly
- Level 1 weekly
- Level 2 daily

### 3.3 Level-0 (full) dump

```bash
# Dump /boot (ext4) to a file
sudo dump -0uf /backup/boot-level0-$(date +%Y%m%d).dump /boot

# Flags:
# -0  = level 0 (full dump)
# -u  = update /etc/dumpdates after successful dump
# -f  = output file (or device)

# Dump to a remote host via SSH
sudo dump -0uf - /boot | ssh backup-server "cat > /backup/boot-level0-$(date +%Y%m%d).dump"
```

### 3.4 The `/etc/dumpdates` file

`dump -u` writes a record to `/etc/dumpdates` after each successful dump. This file tracks when each level was last run, which `dump` uses to determine what has changed.

```bash
# View dump history
sudo cat /etc/dumpdates
# Example output:
# /dev/sda1 0 Mon Feb 17 02:00:00 2026
# /dev/sda1 1 Mon Feb 24 02:00:00 2026
```

### 3.5 Incremental dump (level 1+)

```bash
# Level-1 incremental (changes since last level-0)
sudo dump -1uf /backup/boot-level1-$(date +%Y%m%d).dump /boot

# Level-2 incremental (changes since last level-1)
sudo dump -2uf /backup/boot-level2-$(date +%Y%m%d).dump /boot
```

### 3.6 Dump options reference

| Flag | Description |
|------|-------------|
| `-0` to `-9` | Dump level |
| `-u` | Update `/etc/dumpdates` |
| `-f FILE` | Output to file or device |
| `-z[N]` | Compress output with zlib (level 1-9) |
| `-j[N]` | Compress with bzip2 |
| `-y` | Compress with lzo |
| `-e INODE` | Exclude inode |
| `-n` | Notify users in the `operator` group |
| `-s FEET` | Set tape size |
| `-b SIZE` | Block size in KB |
| `-v` | Verbose output |

### 3.7 Restoring with `restore`

#### Interactive restore (recommended)

The interactive mode lets you browse the dump and select files:

```bash
# Start interactive restore from a dump file
sudo restore -if /backup/boot-level0-20260224.dump

# Interactive commands:
# ls          — list current directory
# cd DIR      — change directory
# add FILE    — mark file for restore
# add DIR     — mark directory for restore
# delete FILE — unmark
# extract     — restore all marked files
# quit        — exit
# verbose     — toggle verbose mode
# help        — show all commands
```

#### Restore entire filesystem

```bash
# Create restore destination and restore
sudo mkdir -p /restore/boot
cd /restore/boot
sudo restore -rf /backup/boot-level0-20260224.dump

# If you have incrementals, apply them in order:
sudo restore -rf /backup/boot-level1-20260225.dump
sudo restore -rf /backup/boot-level2-20260226.dump
```

#### Restore a single file

```bash
cd /restore/boot
sudo restore -xf /backup/boot-level0-20260224.dump ./grub2/grub.cfg
# -x = extract specific files
```

#### Restore to original location (overwrite)

> **⚠️ Danger:** this overwrites the live filesystem in place. Only do this from rescue media or on a freshly created filesystem — never on a running system you cannot afford to corrupt.

```bash
cd /
sudo restore -rf /backup/boot-level0-20260224.dump
```

[↑ Table of Contents](#table-of-contents)

---

## 4. Part B: `xfsdump` and `xfsrestore` (for XFS — RHEL 10 default)

`xfsdump` is the native backup tool for XFS filesystems. It is the correct tool to use for all XFS partitions in RHEL 10.

### 4.1 Installing xfsdump

```bash
sudo dnf install -y xfsdump

# Verify
xfsdump --version
```

### 4.2 xfsdump concepts

| Concept | Description |
|---------|-------------|
| **Session** | A single xfsdump run; identified by a UUID |
| **Dump level** | 0–9 (same concept as dump) |
| **Inventory** | Database of all xfsdump sessions at `/var/lib/xfsdump/inventory/` |
| **Media file** | The output file or device containing the dump |
| **Stream** | xfsdump can write multiple parallel streams |

### 4.3 Level-0 (full) dump of an XFS filesystem

```bash
# Full dump of the root filesystem
sudo xfsdump -l 0 -f /backup/root-level0-$(date +%Y%m%d).xfsdump /

# Full dump of /home
sudo xfsdump -l 0 -f /backup/home-level0-$(date +%Y%m%d).xfsdump /home

# Flags:
# -l 0  = level 0 (full dump)
# -f    = output file or device
```

> **Note:** dumping a busy mounted filesystem captures it mid-flight. For consistent backups of active filesystems, freeze first (Section 6) or dump from an LVM snapshot (Module 05).

During the first run, xfsdump prompts for:
- **Session label** — a human-readable name for this backup (e.g., `root-full-20260224`)
- **Media label** — a label for the output file/device (e.g., `root-media-001`)

To run non-interactively:

```bash
sudo xfsdump -l 0 \
  -L "root-full-$(date +%Y%m%d)" \
  -M "root-media-$(date +%Y%m%d)" \
  -f /backup/root-level0-$(date +%Y%m%d).xfsdump \
  /

# -L = session label
# -M = media label
```

### 4.4 xfsdump options reference

| Flag | Description |
|------|-------------|
| `-l N` | Dump level (0–9) |
| `-f FILE` | Output file or device |
| `-L LABEL` | Session label |
| `-M LABEL` | Media label |
| `-s PATH` | Dump only a subtree (subdirectory) of the filesystem |
| `-I` | Display inventory information |
| `-J` | Inhibit update of inventory (for testing) |
| `-e` | Allow files to be excluded from the dump (via extended attributes) |
| `-v VERBOSITY` | Verbosity: silent, verbose, trace |
| `-p SECS` | Progress report interval in seconds |
| `-b SIZE` | Block size |
| `-z SIZE` | Maximum file size to include |
| `--` | Signals end of options |

### 4.5 Incremental dumps

```bash
# Level-1 incremental (changes since last level-0)
sudo xfsdump -l 1 \
  -L "root-incr1-$(date +%Y%m%d)" \
  -M "root-media-incr1-$(date +%Y%m%d)" \
  -f /backup/root-level1-$(date +%Y%m%d).xfsdump \
  /

# Level-2 incremental
sudo xfsdump -l 2 \
  -L "root-incr2-$(date +%Y%m%d)" \
  -M "root-media-incr2-$(date +%Y%m%d)" \
  -f /backup/root-level2-$(date +%Y%m%d).xfsdump \
  /
```

xfsdump automatically reads its **inventory** (`/var/lib/xfsdump/inventory/`) to determine what changed since the last run of the appropriate level.

### 4.6 Viewing the xfsdump inventory

```bash
# Show all recorded dump sessions
sudo xfsdump -I

# Example output:
# file system 0:
#         fs id:          a1b2c3d4-...
#         session 0:
#                 mount point:    /
#                 device:         /dev/mapper/rhel-root
#                 time:           Mon Feb 17 02:00:00 2026
#                 session label:  "root-full-20260217"
#                 session id:     uuid-here
#                 level:          0
#                 ...
```

### 4.7 Estimating dump size

`xfsdump` has no estimate-only flag — it prints an `estimated dump size` line at the start of every run. To see the estimate without writing a real dump file or polluting the inventory:

```bash
# Dump to /dev/null with -J (don't record in inventory); note the
# "estimated dump size" line near the start of the output, then Ctrl-C
sudo xfsdump -J -l 0 -f /dev/null /

# Quick approximation without xfsdump
sudo du -sxh /
```

### 4.8 Dumping over SSH (piped to remote)

```bash
# Pipe xfsdump output through SSH to remote server
sudo xfsdump -l 0 \
  -L "root-full-$(date +%Y%m%d)" \
  -M "root-media" \
  -f - \
  / | ssh backup-server "cat > /backup/root-level0-$(date +%Y%m%d).xfsdump"
```

[↑ Table of Contents](#table-of-contents)

---

## 5. Restoring with `xfsrestore`

### 5.1 Interactive restore (recommended)

```bash
# Interactive restore mode
sudo xfsrestore -i \
  -f /backup/root-level0-20260224.xfsdump \
  /restore/

# Interactive commands (same as restore):
# ls, cd, add, delete, extract, quit
```

### 5.2 Restore entire filesystem

```bash
# Restore to alternate location
sudo mkdir -p /restore/root
sudo xfsrestore -f /backup/root-level0-20260224.xfsdump /restore/root/

# Restore with verbose output
sudo xfsrestore -v verbose -f /backup/root-level0-20260224.xfsdump /restore/root/
```

### 5.3 Applying incremental restores in order

Chained restores require **cumulative mode (`-r`)** on every step. `xfsrestore -r` keeps a housekeeping directory (`xfsrestorehousekeepingdir`) in the destination to track state between steps.

```bash
# 1. Restore the level-0 full dump first (cumulative mode)
sudo xfsrestore -r -f /backup/root-level0-20260217.xfsdump /restore/root/

# 2. Apply level-1 incremental
sudo xfsrestore -r -f /backup/root-level1-20260218.xfsdump /restore/root/

# 3. Apply level-2 incremental (if any)
sudo xfsrestore -r -f /backup/root-level2-20260219.xfsdump /restore/root/

# 4. After the final increment, remove the housekeeping directory
sudo rm -rf /restore/root/xfsrestorehousekeepingdir
```

### 5.4 Restore a subtree (directory)

```bash
# Restore only /etc from a full root dump
sudo xfsrestore -s etc \
  -f /backup/root-level0-20260224.xfsdump \
  /restore/

# Files will be at /restore/etc/
```

### 5.5 Restore a single file

```bash
# Interactive mode is easiest for single files
sudo xfsrestore -i -f /backup/root-level0-20260224.xfsdump /restore/
# Then: cd etc/ssh; add sshd_config; extract; quit
```

### 5.6 xfsrestore options reference

| Flag | Description |
|------|-------------|
| `-f FILE` | Input file or device |
| `-i` | Interactive mode |
| `-s PATH` | Restore only a subtree |
| `-r` | Cumulative mode — apply a level-0 + incremental chain in sequence |
| `-R` | Resume an interrupted restore |
| `-S UUID` | Restore specific session by UUID |
| `-v VERBOSITY` | Verbosity level |
| `-p SECS` | Progress interval |

[↑ Table of Contents](#table-of-contents)

---

## 6. XFS Freeze for Consistent Live Backups

When backing up a mounted XFS filesystem, you can freeze it momentarily for consistency:

```bash
# Freeze the filesystem (writes are paused, reads continue)
sudo xfs_freeze -f /home

# Take your backup (LVM snapshot, xfsdump, etc.)
sudo xfsdump -l 0 -L "home-frozen" -M "home-media" -f /backup/home.xfsdump /home

# Unfreeze — I/O resumes
sudo xfs_freeze -u /home
```

**Important:** Keep the freeze window as short as possible. Applications waiting on I/O will block during the freeze.

[↑ Table of Contents](#table-of-contents)

---

## 7. Comparing dump/xfsdump Approaches

### When to use `xfsdump`

- Backing up large XFS filesystems (root, /home, /var)
- You need native level-based incrementals with automatic inventory tracking
- You need to back up to tape or pipe to a remote host
- You want the fastest possible full filesystem backup

### When to use `tar` or `rsync` instead

- You need portability (tar archives can be read anywhere)
- You are backing up a subset of files, not a whole filesystem
- You need encrypted archives (combine xfsdump with GPG — see Module 11)
- You need deduplication across backups

### Tool decision matrix

| Need | Recommended Tool |
|------|-----------------|
| Fast full XFS filesystem backup | `xfsdump` |
| Incremental XFS backup with level system | `xfsdump` |
| Portable archives across systems | `tar` |
| Space-efficient incrementals (hardlinks) | `rsync --link-dest` |
| Encrypted remote backups | `restic` |
| Enterprise scheduling and catalog | `bareos` |

[↑ Table of Contents](#table-of-contents)

---

## 8. Automation Script

```bash
sudo tee /usr/local/bin/xfsdump-backup.sh <<'SCRIPT'
#!/bin/bash
# xfsdump-backup.sh — XFS filesystem backup with level rotation
# Usage: xfsdump-backup.sh [0|1|2]  (default: 1)

set -euo pipefail

BACKUP_DIR="/backup/xfsdump"
LOG_FILE="/var/log/xfsdump-backup.log"
LEVEL="${1:-1}"
DATE=$(date +%Y%m%d)
HOST=$(hostname -s)

# Map: mountpoint -> output name
declare -A FILESYSTEMS
FILESYSTEMS["/"]="root"
FILESYSTEMS["/home"]="home"
FILESYSTEMS["/var"]="var"
FILESYSTEMS["/boot"]="boot"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "${LOG_FILE}"; }

mkdir -p "${BACKUP_DIR}"

log "=== xfsdump START: level=${LEVEL} host=${HOST} ==="

for MOUNT in "${!FILESYSTEMS[@]}"; do
  NAME="${FILESYSTEMS[$MOUNT]}"
  OUTFILE="${BACKUP_DIR}/${HOST}-${NAME}-level${LEVEL}-${DATE}.xfsdump"

  # Check if mountpoint exists and is XFS
  FSTYPE=$(df -T "${MOUNT}" 2>/dev/null | awk 'NR==2{print $2}')
  if [[ "${FSTYPE}" != "xfs" ]]; then
    log "SKIP: ${MOUNT} is ${FSTYPE}, not xfs"
    continue
  fi

  log "Dumping ${MOUNT} (level ${LEVEL}) -> ${OUTFILE}"
  xfsdump \
    -l "${LEVEL}" \
    -L "${HOST}-${NAME}-level${LEVEL}-${DATE}" \
    -M "${HOST}-${NAME}-media-${DATE}" \
    -f "${OUTFILE}" \
    "${MOUNT}" \
    2>>"${LOG_FILE}" \
    && log "OK: ${OUTFILE} ($(du -sh "${OUTFILE}" | cut -f1))" \
    || log "ERROR: ${MOUNT} dump failed"
done

# Clean up old dumps (keep 14 days)
log "Cleaning dumps older than 14 days"
find "${BACKUP_DIR}" -name "*.xfsdump" -mtime +14 -delete

log "=== xfsdump END ==="
SCRIPT

sudo chmod +x /usr/local/bin/xfsdump-backup.sh
```

[↑ Table of Contents](#table-of-contents)

---

## Lab Exercises

### Lab 04-1: Identify your filesystem types

```bash
# 1. List all mounted filesystems and their types
df -T

# 2. Specifically check root and /home
df -T / /home /boot /var

# 3. Identify which tool to use for each
# XFS -> xfsdump/xfsrestore
# ext4 -> dump/restore (reference only — package not available on RHEL 10, see §3.1)
```

### Lab 04-2: Full xfsdump of /boot or /home

```bash
# 1. Check /boot filesystem type
df -T /boot

# 2. If XFS: use xfsdump
sudo xfsdump -l 0 \
  -L "boot-full-$(date +%Y%m%d)" \
  -M "boot-media-001" \
  -f /backup/boot-level0-$(date +%Y%m%d).xfsdump \
  /boot

# 3. View the inventory
sudo xfsdump -I

# 4. Check output file size
ls -lh /backup/boot-level0-$(date +%Y%m%d).xfsdump
```

### Lab 04-3: Make changes and run incremental

```bash
# 1. Create some test files in /home
echo "lab test file $(date)" | sudo tee /home/xfsdump-test.txt

# 2. Run level-1 incremental
sudo xfsdump -l 1 \
  -L "home-incr1-$(date +%Y%m%d)" \
  -M "home-media-incr1" \
  -f /backup/home-level1-$(date +%Y%m%d).xfsdump \
  /home

# 3. Verify the incremental is smaller than a full
ls -lh /backup/home-level*.xfsdump 2>/dev/null || echo "No level-0 exists yet — run level-0 first"
```

### Lab 04-4: Interactive restore

```bash
# 1. Create a restore directory
sudo mkdir -p /restore/xfs_test

# 2. Launch interactive restore
sudo xfsrestore -i \
  -f /backup/home-level1-$(date +%Y%m%d).xfsdump \
  /restore/xfs_test/
# Inside interactive mode:
# > ls
# > cd /
# > add home
# > extract
# > quit

# 3. Verify the restored files
ls /restore/xfs_test/
```

### Lab 04-5: Run the automated backup script

```bash
# Full backup (level 0)
sudo /usr/local/bin/xfsdump-backup.sh 0

# View log
sudo cat /var/log/xfsdump-backup.log

# View inventory
sudo xfsdump -I
```

[↑ Table of Contents](#table-of-contents)

---

## Review Questions

1. What is the key difference between `dump` and `xfsdump`? Which should you use on RHEL 10?
2. What does "level 0" mean in the dump level system?
3. What does a level-2 dump back up relative to a level-1 dump?
4. What file does `dump -u` write to, and why is it important?
5. What is the xfsdump inventory, and where is it stored?
6. When restoring incremental dumps, in what order must they be applied?
7. What does `xfs_freeze` do, and why is it useful for backups?
8. What flag limits an `xfsrestore` to a specific subdirectory?
9. How do you estimate the size of an xfsdump before running it?
10. What command lets you browse the contents of an xfsdump archive interactively?

[↑ Table of Contents](#table-of-contents)

---

## Answers to Review Questions

1. `dump` works on **ext2/ext3/ext4** filesystems only. `xfsdump` works on **XFS** only. Since RHEL 10 defaults to XFS for all partitions, `xfsdump`/`xfsrestore` should be used for almost all RHEL 10 backup operations.
2. Level 0 is a **complete dump** of the entire filesystem — all files regardless of when they were last modified. It is the baseline from which all incremental levels build.
3. A level-2 dump backs up all files changed since the **last level-1 dump** (not the last level-0). Each level N backs up changes since the last dump of a lower level.
4. `dump -u` writes to `/etc/dumpdates`. This file records the filesystem, dump level, and timestamp of each successful dump. It allows subsequent incremental dumps to know exactly when the last backup of each level occurred.
5. The **xfsdump inventory** is a database of all xfsdump sessions, stored at `/var/lib/xfsdump/inventory/`. It records session UUIDs, timestamps, levels, and source filesystems, enabling xfsdump to automatically determine what changed since the last backup.
6. Incrementals must be applied in **chronological order starting with the level-0 full dump**, using cumulative mode: `xfsrestore -r` for every step (level-0 first, then level-1, then level-2). Remove the `xfsrestorehousekeepingdir` from the destination after the final increment.
7. `xfs_freeze -f MOUNTPOINT` **pauses all write I/O** to an XFS filesystem while keeping reads active. This provides a crash-consistent state for backup. It is used to create a point-in-time freeze before taking a dump or snapshot. `xfs_freeze -u` unfreezes.
8. `xfsrestore -s PATH` — restores only the subtree starting at `PATH` relative to the filesystem root.
9. xfsdump has no estimate-only flag (`-e` means "allow excludes"). Every run prints an `estimated dump size` line at the start — run `xfsdump -J -l LEVEL -f /dev/null MOUNTPOINT`, read the estimate, and abort; or approximate with `du -sx MOUNTPOINT`.
10. `xfsrestore -i -f DUMPFILE DESTDIR` — launches interactive mode where you can browse, select, and extract files.

[↑ Table of Contents](#table-of-contents)

---

*Previous: [03 — rsync](03-rsync.md)*
*Next: [05 — LVM Snapshots](05-lvm-snapshots.md)*

© 2026 UncleJS — Licensed under CC BY-NC-SA 4.0
