# Module 02 — `tar` — Archives, Compression & Incrementals
[![CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey)](./LICENSE.md)
[![RHEL 10](https://img.shields.io/badge/platform-RHEL%2010-red)](https://access.redhat.com/products/red-hat-enterprise-linux)
[![Backup](https://img.shields.io/badge/Backup-RHEL%2010-blue)](https://access.redhat.com/products/red-hat-enterprise-linux)

## Learning Objectives

By the end of this module you will be able to:

- Create full and compressed tar archives of any directory tree
- Choose the right compression algorithm for your use case
- Perform incremental backups with `--listed-incremental`
- Restore an entire archive, a single file, or a directory from a tar backup
- Preserve file permissions, ownership, SELinux labels, and ACLs
- Automate tar backups and manage naming conventions

---

## Table of Contents

- [1. What is `tar`?](#1-what-is-tar)
- [2. Basic Concepts](#2-basic-concepts)
  - [Archive vs Compression](#archive-vs-compression)
  - [Common File Extensions](#common-file-extensions)
- [3. Compression Algorithm Comparison](#3-compression-algorithm-comparison)
  - [Recommendation for RHEL 10 backups](#recommendation-for-rhel-10-backups)
- [4. Essential `tar` Flags Reference](#4-essential-tar-flags-reference)
- [5. Creating Full Backups](#5-creating-full-backups)
  - [5.1 Basic full backup](#51-basic-full-backup)
  - [5.2 Full backup with metadata preservation (recommended)](#52-full-backup-with-metadata-preservation-recommended)
  - [5.3 Full system backup (multiple directories)](#53-full-system-backup-multiple-directories)
  - [5.4 Using an exclude file](#54-using-an-exclude-file)
- [6. Listing and Verifying Archives](#6-listing-and-verifying-archives)
- [7. Extracting (Restoring) from Archives](#7-extracting-restoring-from-archives)
  - [7.1 Extract entire archive to original location](#71-extract-entire-archive-to-original-location)
  - [7.2 Extract to an alternate location](#72-extract-to-an-alternate-location)
  - [7.3 Extract a single file](#73-extract-a-single-file)
  - [7.4 Extract a single directory](#74-extract-a-single-directory)
  - [7.5 Extract with verbose to see what's being restored](#75-extract-with-verbose-to-see-whats-being-restored)
  - [7.6 Preview what would be extracted (dry run)](#76-preview-what-would-be-extracted-dry-run)
- [8. Incremental Backups with `--listed-incremental`](#8-incremental-backups-with---listed-incremental)
  - [How it works](#how-it-works)
  - [8.1 Level-0: Full backup](#81-level-0-full-backup)
  - [8.2 Level-1: Incremental backup](#82-level-1-incremental-backup)
  - [8.3 Preserving snapshot state for a full cycle](#83-preserving-snapshot-state-for-a-full-cycle)
  - [8.4 Restoring an incremental backup chain](#84-restoring-an-incremental-backup-chain)
- [9. Handling Deleted Files in Incrementals](#9-handling-deleted-files-in-incrementals)
- [10. Splitting Large Archives](#10-splitting-large-archives)
- [11. Streaming tar Over SSH](#11-streaming-tar-over-ssh)
- [12. Naming Conventions](#12-naming-conventions)
- [13. Complete Backup Script](#13-complete-backup-script)
- [Lab Exercises](#lab-exercises)
  - [Lab 02-1: Create and verify a basic archive](#lab-02-1-create-and-verify-a-basic-archive)
  - [Lab 02-2: SELinux label preservation](#lab-02-2-selinux-label-preservation)
  - [Lab 02-3: Incremental backup cycle](#lab-02-3-incremental-backup-cycle)
  - [Lab 02-4: Full restore from incremental chain](#lab-02-4-full-restore-from-incremental-chain)
  - [Lab 02-5: Backup /etc with the complete production script](#lab-02-5-backup-etc-with-the-complete-production-script)
- [Review Questions](#review-questions)
- [Answers to Review Questions](#answers-to-review-questions)

---

## 1. What is `tar`?

`tar` (Tape ARchive) is the oldest and most universally available backup utility on Unix/Linux systems. Despite its age, it remains one of the most powerful tools for:

- Creating portable archives of directory trees
- Preserving all file metadata (permissions, ownership, timestamps)
- Supporting multiple compression algorithms
- Performing incremental backups via snapshot files

`tar` is available on every RHEL 10 system with no additional packages required.

```bash
# Verify tar is installed and check version
tar --version
```

[↑ Table of Contents](#table-of-contents)

---

## 2. Basic Concepts

### Archive vs Compression

`tar` does two separate things:

1. **Archiving** — combining many files/directories into a single stream or file (`.tar`)
2. **Compression** — reducing the size of that stream (`.gz`, `.bz2`, `.xz`, `.zst`)

These are separate operations. You can have an uncompressed tar archive (`.tar`) or a compressed one (`.tar.gz`, `.tar.bz2`, etc.).

### Common File Extensions

| Extension | Description |
|-----------|-------------|
| `.tar` | Uncompressed archive |
| `.tar.gz` or `.tgz` | gzip compressed |
| `.tar.bz2` or `.tbz2` | bzip2 compressed |
| `.tar.xz` or `.txz` | xz (LZMA2) compressed |
| `.tar.zst` | Zstandard compressed |

[↑ Table of Contents](#table-of-contents)

---

## 3. Compression Algorithm Comparison

| Algorithm | Flag | Speed | Ratio | Best For |
|-----------|------|-------|-------|----------|
| gzip | `-z` | Fast | Medium | General use, fast backups |
| bzip2 | `-j` | Medium | Good | Better compression, less speed-sensitive |
| xz | `-J` | Slow | Excellent | Long-term archival, maximum compression |
| zstd | `--zstd` | Very fast | Very good | Modern systems, best speed/ratio balance |
| none | — | Fastest | None | Piping to another tool, NFS/raw storage |

### Recommendation for RHEL 10 backups

- **Daily backups:** `gzip` or `zstd` — speed matters
- **Weekly/monthly archives:** `xz` — maximum compression
- **Piping over SSH or network:** no compression in `tar` — let SSH handle it, or use `zstd`

[↑ Table of Contents](#table-of-contents)

---

## 4. Essential `tar` Flags Reference

| Flag | Long Form | Description |
|------|-----------|-------------|
| `-c` | `--create` | Create a new archive |
| `-x` | `--extract` | Extract files from archive |
| `-t` | `--list` | List contents of archive |
| `-v` | `--verbose` | Show filenames being processed |
| `-f` | `--file=` | Specify archive filename |
| `-z` | `--gzip` | Compress with gzip |
| `-j` | `--bzip2` | Compress with bzip2 |
| `-J` | `--xz` | Compress with xz |
| `--zstd` | `--zstd` | Compress with Zstandard |
| `-p` | `--preserve-permissions` | Preserve all permissions |
| `-P` | `--absolute-names` | Keep leading `/` in paths |
| `--xattrs` | `--xattrs` | Preserve extended attributes (SELinux labels) |
| `--acls` | `--acls` | Preserve POSIX ACLs |
| `--selinux` | `--selinux` | Preserve SELinux contexts |
| `-g` | `--listed-incremental=` | Incremental backup snapshot file |
| `--exclude=` | `--exclude=` | Exclude a path or pattern |
| `--exclude-from=` | `--exclude-from=` | Read exclusions from a file |
| `-C` | `--directory=` | Change to directory before extracting |
| `--numeric-owner` | | Use numeric UID/GID (portable) |
| `-d` | `--compare` | Compare archive contents against the filesystem (verify) |
| `-h` | `--dereference` | Follow symlinks |

[↑ Table of Contents](#table-of-contents)

---

## 5. Creating Full Backups

### 5.1 Basic full backup

```bash
# Create a compressed backup of /etc
sudo tar -czf /backup/etc-$(date +%Y%m%d).tar.gz /etc

# With verbose output
sudo tar -czvf /backup/etc-$(date +%Y%m%d).tar.gz /etc
```

### 5.2 Full backup with metadata preservation (recommended)

For RHEL 10 systems with SELinux and ACLs, always include these flags:

```bash
sudo tar \
  --create \
  --file=/backup/etc-$(date +%Y%m%d-%H%M).tar.gz \
  --gzip \
  --preserve-permissions \
  --numeric-owner \
  --xattrs \
  --acls \
  --selinux \
  /etc/

# Shorthand version
sudo tar -czpf /backup/etc-full-$(date +%Y%m%d).tar.gz \
  --xattrs --acls --selinux --numeric-owner \
  /etc/
```

**Why `--numeric-owner`?** UIDs and GIDs by name can differ across machines. Storing numeric IDs ensures correct ownership after restore even if the user database differs.

### 5.3 Full system backup (multiple directories)

```bash
sudo tar -czpf /backup/system-full-$(date +%Y%m%d).tar.gz \
  --xattrs --acls --selinux --numeric-owner \
  --exclude=/proc \
  --exclude=/sys \
  --exclude=/dev \
  --exclude=/run \
  --exclude=/tmp \
  --exclude=/var/tmp \
  --exclude=/var/cache \
  --exclude=/media \
  --exclude=/mnt \
  --exclude=/backup \
  /
```

### 5.4 Using an exclude file

For complex exclusion lists, use a file:

```bash
# Create exclusions file
sudo tee /etc/backup/tar-exclude.txt <<'EOF'
/proc
/sys
/dev
/run
/tmp
/var/tmp
/var/cache
/media
/mnt
/backup
/var/lib/containers/storage/overlay
EOF

# Use it in the tar command
sudo tar -czpf /backup/system-full-$(date +%Y%m%d).tar.gz \
  --xattrs --acls --selinux --numeric-owner \
  --exclude-from=/etc/backup/tar-exclude.txt \
  /
```

[↑ Table of Contents](#table-of-contents)

---

## 6. Listing and Verifying Archives

```bash
# List contents of an archive
tar -tzf /backup/etc-20260224.tar.gz

# List with full details (permissions, size, timestamp)
tar -tzvf /backup/etc-20260224.tar.gz

# Verify archive integrity (reads and checks structure)
tar -tzf /backup/etc-20260224.tar.gz > /dev/null && echo "Archive OK" || echo "Archive CORRUPT"

# Compare archive to current filesystem
sudo tar --compare --file=/backup/etc-20260224.tar.gz --gzip /etc/
# Output shows files that differ since backup was taken
```

[↑ Table of Contents](#table-of-contents)

---

## 7. Extracting (Restoring) from Archives

### 7.1 Extract entire archive to original location

```bash
# CAUTION: This overwrites existing files
sudo tar -xzpf /backup/etc-20260224.tar.gz \
  --xattrs --acls --selinux --numeric-owner \
  -C /
# -C / means "extract relative to /" so /etc/passwd restores to /etc/passwd
```

### 7.2 Extract to an alternate location

```bash
# Extract to /restore instead of original paths
sudo mkdir -p /restore
sudo tar -xzpf /backup/etc-20260224.tar.gz \
  --xattrs --acls --selinux --numeric-owner \
  -C /restore

# The archive's /etc will be at /restore/etc/
ls /restore/etc/
```

### 7.3 Extract a single file

```bash
# Extract just /etc/ssh/sshd_config from the archive
sudo tar -xzf /backup/etc-20260224.tar.gz \
  -C / \
  etc/ssh/sshd_config
# Note: path inside archive has no leading /
```

### 7.4 Extract a single directory

```bash
# Extract just the /etc/nginx/ directory
sudo tar -xzf /backup/etc-20260224.tar.gz \
  -C / \
  etc/nginx/
```

### 7.5 Extract with verbose to see what's being restored

```bash
sudo tar -xzvpf /backup/etc-20260224.tar.gz \
  --xattrs --acls --selinux --numeric-owner \
  -C /restore
```

### 7.6 Preview what would be extracted (dry run)

```bash
# List matching files without extracting
tar -tzf /backup/etc-20260224.tar.gz | grep sshd
```

[↑ Table of Contents](#table-of-contents)

---

## 8. Incremental Backups with `--listed-incremental`

`tar`'s `--listed-incremental` (or `-g`) option enables level-0 (full) and level-N (incremental) backups using a **snapshot file** that records the state of the directory.

### How it works

1. **Level-0 (Full):** Run with `-g /path/to/snapshot.snar` and a new or empty snapshot file. `tar` records all files and their metadata in the snapshot file and creates a full archive.
2. **Level-1+ (Incremental):** Run again with `-g` pointing to the **same** snapshot file. `tar` compares current state to snapshot, backs up only changed/new files, and updates the snapshot.

### 8.1 Level-0: Full backup

```bash
# Remove old snapshot to force a new full backup
sudo rm -f /backup/snapshots/etc.snar

# Create the full backup with snapshot
sudo mkdir -p /backup/snapshots
sudo tar -czpf /backup/etc-level0-$(date +%Y%m%d).tar.gz \
  --xattrs --acls --selinux --numeric-owner \
  --listed-incremental=/backup/snapshots/etc.snar \
  /etc/

echo "Level-0 complete. Snapshot saved at /backup/snapshots/etc.snar"
```

### 8.2 Level-1: Incremental backup

```bash
# Run without removing the snapshot file
# tar reads the snapshot, backs up only changes, and updates the snapshot
sudo tar -czpf /backup/etc-incr-$(date +%Y%m%d-%H%M).tar.gz \
  --xattrs --acls --selinux --numeric-owner \
  --listed-incremental=/backup/snapshots/etc.snar \
  /etc/

# The .snar file now reflects the current state
```

**⚠️ Important:** The snapshot file (`.snar`) is updated in place after every run. To correctly perform incrementals, you need to:
1. Start each week (or cycle) with a fresh `.snar` by **removing it** (or pointing `--listed-incremental` at a new path) before the full backup — copying it does *not* reset the chain
2. Never modify the `.snar` between incrementals

### 8.3 Preserving snapshot state for a full cycle

Best practice: keep a copy of the snapshot at the time of the full backup, and use copies for incrementals.

```bash
# Weekly full backup script pattern:
SNAPDIR=/backup/snapshots
BACKUPDIR=/backup/weekly

# Remove old snapshot, take full backup
sudo rm -f ${SNAPDIR}/etc.snar
sudo tar -czpf ${BACKUPDIR}/etc-full-$(date +%Y%m%d).tar.gz \
  --xattrs --acls --selinux --numeric-owner \
  --listed-incremental=${SNAPDIR}/etc.snar \
  /etc/

# Save a copy of the snapshot for reference
sudo cp ${SNAPDIR}/etc.snar ${SNAPDIR}/etc.snar.full-$(date +%Y%m%d)
```

**Note:** the `.snar` file is required state for *creating* the next incremental in the chain. Restores work without it, but if `/backup/snapshots` is lost, the next "incremental" silently becomes a new level-0. Protect the snapshot directory (e.g., include it in the backup set).

### 8.4 Restoring an incremental backup chain

To restore from an incremental chain, you **must apply archives in order**:

```bash
# Step 1: Restore the level-0 full backup
sudo tar -xzpf /backup/etc-level0-20260217.tar.gz \
  --xattrs --acls --selinux --numeric-owner \
  --listed-incremental=/dev/null \
  -C /restore

# Step 2: Apply incremental backups in chronological order
sudo tar -xzpf /backup/etc-incr-20260218.tar.gz \
  --xattrs --acls --selinux --numeric-owner \
  --listed-incremental=/dev/null \
  -C /restore

sudo tar -xzpf /backup/etc-incr-20260219.tar.gz \
  --xattrs --acls --selinux --numeric-owner \
  --listed-incremental=/dev/null \
  -C /restore

# Continue for each incremental until you reach the desired restore point
```

**Note:** `--listed-incremental=/dev/null` tells tar to apply the incremental without needing the original snapshot (just apply all changes in the archive, including deletions).

[↑ Table of Contents](#table-of-contents)

---

## 9. Handling Deleted Files in Incrementals

When `tar` creates an incremental backup, it records files that were **deleted** since the last run. When you apply the incremental with `--listed-incremental=/dev/null`, `tar` will remove those files from the restore destination.

This is how `tar` incrementals handle file deletion — it is automatic and correct.

[↑ Table of Contents](#table-of-contents)

---

## 10. Splitting Large Archives

For very large backups, split into manageable chunks:

```bash
# Create and split: pipe through split
sudo tar -czp \
  --xattrs --acls --selinux --numeric-owner \
  --exclude-from=/etc/backup/tar-exclude.txt \
  / | split -b 4G - /backup/system-$(date +%Y%m%d).tar.gz.part-

# Files created: system-20260224.tar.gz.part-aa, part-ab, etc.

# Reassemble and extract
cat /backup/system-20260224.tar.gz.part-* | sudo tar -xzpf - \
  --xattrs --acls --selinux --numeric-owner \
  -C /restore
```

[↑ Table of Contents](#table-of-contents)

---

## 11. Streaming tar Over SSH

Send a backup directly to a remote machine without creating a local copy:

```bash
# Backup /etc to remote server via SSH
sudo tar -czp \
  --xattrs --acls --selinux --numeric-owner \
  /etc | ssh backup-server "cat > /backup/clients/$(hostname)/etc-$(date +%Y%m%d).tar.gz"

# Or using tar's remote option
sudo tar -czpf - \
  --xattrs --acls --selinux --numeric-owner \
  /etc | ssh backup-server "tar -xzpf - --xattrs --acls --selinux --numeric-owner -C /restore/$(hostname)/etc"
```

[↑ Table of Contents](#table-of-contents)

---

## 12. Naming Conventions

Consistent naming prevents confusion and enables automation:

```
PATTERN:  {hostname}-{source}-{type}-{date}-{time}.tar.{ext}

Examples:
  backup-client-etc-full-20260224-0200.tar.gz
  backup-client-etc-incr-20260225-0200.tar.gz
  backup-client-home-full-20260217-0300.tar.xz
  backup-server-varlib-full-20260224-0100.tar.gz
```

```bash
# Use in scripts:
HOST=$(hostname -s)
DATE=$(date +%Y%m%d)
TIME=$(date +%H%M)
SOURCE="etc"
TYPE="full"

FILENAME="${HOST}-${SOURCE}-${TYPE}-${DATE}-${TIME}.tar.gz"
sudo tar -czpf /backup/${FILENAME} --xattrs --acls --selinux /etc/
```

[↑ Table of Contents](#table-of-contents)

---

## 13. Complete Backup Script

A production-ready daily tar backup script:

```bash
sudo tee /usr/local/bin/tar-backup.sh <<'SCRIPT'
#!/bin/bash
# tar-backup.sh — Daily full backup of critical RHEL 10 paths
# Usage: tar-backup.sh [full|incr]

set -euo pipefail

### Configuration ###
BACKUP_ROOT="/backup/tar"
SNAPSHOT_DIR="/backup/snapshots"
LOG_FILE="/var/log/tar-backup.log"
RETENTION_DAYS=14
HOST=$(hostname -s)
DATE=$(date +%Y%m%d)
TIME=$(date +%H%M)
MODE="${1:-full}"   # Default to full if no argument

SOURCES=(
  "/etc"
  "/root"
  "/home"
  "/boot"
  "/var/www"
  "/opt"
  "/srv"
  "/usr/local"
)

EXCLUDES=(
  "--exclude=/proc"
  "--exclude=/sys"
  "--exclude=/dev"
  "--exclude=/run"
  "--exclude=/tmp"
  "--exclude=/var/tmp"
  "--exclude=/var/cache"
  "--exclude=/backup"
)

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "${LOG_FILE}"; }

mkdir -p "${BACKUP_ROOT}" "${SNAPSHOT_DIR}"

log "=== tar backup START: mode=${MODE} host=${HOST} ==="

for SRC in "${SOURCES[@]}"; do
  [ -d "${SRC}" ] || { log "SKIP: ${SRC} not found"; continue; }

  SRCNAME=$(echo "${SRC}" | tr '/' '-' | sed 's/^-//')
  SNAP="${SNAPSHOT_DIR}/${HOST}-${SRCNAME}.snar"
  OUTFILE="${BACKUP_ROOT}/${HOST}-${SRCNAME}-${MODE}-${DATE}-${TIME}.tar.gz"

  if [[ "${MODE}" == "full" ]]; then
    log "Removing snapshot for full backup: ${SNAP}"
    rm -f "${SNAP}"
  fi

  log "Backing up ${SRC} -> ${OUTFILE}"
  tar -czpf "${OUTFILE}" \
    --xattrs --acls --selinux --numeric-owner \
    --listed-incremental="${SNAP}" \
    "${EXCLUDES[@]}" \
    "${SRC}" 2>>"${LOG_FILE}" \
    && log "OK: ${OUTFILE}" \
    || log "ERROR: Failed to back up ${SRC}"
done

# Cleanup old backups
log "Removing backups older than ${RETENTION_DAYS} days"
find "${BACKUP_ROOT}" -name "*.tar.gz" -mtime +${RETENTION_DAYS} -delete

log "=== tar backup END ==="
SCRIPT

sudo chmod +x /usr/local/bin/tar-backup.sh
```

[↑ Table of Contents](#table-of-contents)

---

## Lab Exercises

### Lab 02-1: Create and verify a basic archive

```bash
# 1. Create a test directory with some files
mkdir -p /tmp/tartest/{docs,configs,scripts}
cp /etc/hostname /tmp/tartest/configs/
cp /etc/os-release /tmp/tartest/configs/
echo "test document" > /tmp/tartest/docs/notes.txt
printf '#!/bin/bash\necho hello\n' > /tmp/tartest/scripts/hello.sh
chmod +x /tmp/tartest/scripts/hello.sh

# 2. Create a gzip archive
tar -czpvf /tmp/tartest-$(date +%Y%m%d).tar.gz /tmp/tartest/

# 3. List the archive contents
tar -tzvf /tmp/tartest-$(date +%Y%m%d).tar.gz

# 4. Verify integrity
tar -tzf /tmp/tartest-$(date +%Y%m%d).tar.gz > /dev/null && echo "OK"
```

### Lab 02-2: SELinux label preservation

```bash
# 1. Create a file with a custom SELinux context
sudo touch /tmp/seltest.txt
sudo chcon -t httpd_sys_content_t /tmp/seltest.txt
ls -Z /tmp/seltest.txt

# 2. Archive it with SELinux flag
sudo tar -czpf /tmp/seltest.tar.gz --xattrs --selinux /tmp/seltest.txt

# 3. Extract to alternate location and check labels
sudo tar -xzpf /tmp/seltest.tar.gz --xattrs --selinux -C /tmp/restore_test/
ls -Z /tmp/restore_test/tmp/seltest.txt
# Should show httpd_sys_content_t
```

### Lab 02-3: Incremental backup cycle

```bash
# 1. Set up test source directory
mkdir -p /tmp/incr_source
echo "file1 content" > /tmp/incr_source/file1.txt
echo "file2 content" > /tmp/incr_source/file2.txt
mkdir -p /backup/incr_test /backup/snapshots

# 2. Level-0 full backup
sudo rm -f /backup/snapshots/incr_test.snar
tar -czpf /backup/incr_test/full-$(date +%Y%m%d).tar.gz \
  --listed-incremental=/backup/snapshots/incr_test.snar \
  /tmp/incr_source/
echo "Level-0 done. Snapshot:"
strings /backup/snapshots/incr_test.snar | head -5

# 3. Make changes
echo "file3 NEW" > /tmp/incr_source/file3.txt
echo "file1 MODIFIED" > /tmp/incr_source/file1.txt
rm /tmp/incr_source/file2.txt

# 4. Level-1 incremental backup
tar -czpf /backup/incr_test/incr-$(date +%Y%m%d-%H%M).tar.gz \
  --listed-incremental=/backup/snapshots/incr_test.snar \
  /tmp/incr_source/

# 5. List contents of incremental — should show only file1 and file3
tar -tzvf /backup/incr_test/incr-*.tar.gz
```

### Lab 02-4: Full restore from incremental chain

```bash
# 1. Restore full backup first
mkdir -p /tmp/incr_restore
tar -xzpf /backup/incr_test/full-*.tar.gz \
  --listed-incremental=/dev/null \
  -C /tmp/incr_restore
ls /tmp/incr_restore/tmp/incr_source/

# 2. Apply incremental
tar -xzpf /backup/incr_test/incr-*.tar.gz \
  --listed-incremental=/dev/null \
  -C /tmp/incr_restore
ls /tmp/incr_restore/tmp/incr_source/
# file1.txt should be modified, file2.txt should be gone, file3.txt should appear
```

### Lab 02-5: Backup /etc with the complete production script

```bash
# Install and run the backup script
sudo bash /usr/local/bin/tar-backup.sh full
sudo ls -lh /backup/tar/
sudo cat /var/log/tar-backup.log
```

[↑ Table of Contents](#table-of-contents)

---

## Review Questions

1. What is the difference between archiving and compression in `tar`?
2. Which `tar` flag enables incremental backups, and what does the snapshot file record?
3. In what order must incremental backups be restored, and why?
4. Why should you use `--numeric-owner` when creating backups on RHEL 10?
5. Which three flags must be added to `tar` to fully preserve SELinux labels and ACLs?
6. How do you extract a single file from a tar archive without extracting the whole thing?
7. What happens to deleted files when you restore an incremental backup using `--listed-incremental=/dev/null`?
8. What is the `-C` flag used for during extraction?
9. Compare `gzip`, `xz`, and `zstd` compression — when would you choose each?
10. Why must the backup destination directory be excluded from a full system `tar` backup?

[↑ Table of Contents](#table-of-contents)

---

## Answers to Review Questions

1. **Archiving** = combining files into a single `.tar` stream. **Compression** = reducing the size of that stream. They are independent operations. You can archive without compressing, or compress an already-archived file.
2. `--listed-incremental=FILE` (or `-g FILE`). The snapshot file records file metadata (inode, mtime, size) so `tar` can determine what changed since the last run.
3. In **chronological order** starting with the level-0 full backup. Because each incremental contains only changes since the previous one — missing a link in the chain means some changes won't be applied.
4. UIDs and GIDs by name can differ between machines. `--numeric-owner` stores numeric IDs ensuring correct ownership regardless of whether the same usernames exist on the restore target.
5. `--xattrs`, `--acls`, `--selinux`
6. `tar -xzf archive.tar.gz -C /destination path/inside/archive/file.txt` — specify the relative path of the file as it appears inside the archive (no leading `/`).
7. `tar` removes those files from the restore destination, correctly reconstructing the deleted state. The incremental archive stores deletion records and applies them during restore.
8. `-C /directory` changes the working directory before extracting, so all files are placed relative to that directory rather than the filesystem root.
9. **gzip** — fast, widely compatible, good for daily backups. **xz** — slowest, highest compression ratio, best for long-term archival storage where space is critical. **zstd** — very fast with excellent compression ratio; best balance for modern systems.
10. If the backup destination is inside a path being archived, `tar` will recurse into it and either loop indefinitely or create an archive containing itself, wasting space and potentially corrupting the backup.

[↑ Table of Contents](#table-of-contents)

---

*Previous: [01 — RHEL 10 Filesystem & What to Back Up](01-rhel10-filesystem.md)*
*Next: [03 — rsync](03-rsync.md)*

© 2026 UncleJS — Licensed under CC BY-NC-SA 4.0
