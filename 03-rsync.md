# Module 03 — `rsync` — Local/Remote Sync & Snapshot-Style Incrementals

## Learning Objectives

By the end of this module you will be able to:

- Use rsync for local and remote file synchronisation
- Understand every critical rsync flag and what it does
- Implement snapshot-style incremental backups using `--link-dest`
- Transfer backups securely over SSH
- Configure SSH keys for passwordless automated backups
- Write a production-grade rsync backup script with rotation and alerting
- Understand rsync's push vs pull models and when to use each

---

## Table of Contents

- [1. What is rsync?](#1-what-is-rsync)
- [2. rsync Flag Reference](#2-rsync-flag-reference)
  - [Essential flags](#essential-flags)
  - [Sync behaviour flags](#sync-behaviour-flags)
  - [Metadata and extended attributes](#metadata-and-extended-attributes)
  - [Backup and incremental flags](#backup-and-incremental-flags)
  - [Transfer and safety flags](#transfer-and-safety-flags)
  - [Path and filter flags](#path-and-filter-flags)
- [3. Trailing Slash Gotcha](#3-trailing-slash-gotcha)
- [4. Local Backups](#4-local-backups)
  - [Simple local sync](#simple-local-sync)
  - [Local backup with stats](#local-backup-with-stats)
- [5. Remote Backups over SSH](#5-remote-backups-over-ssh)
  - [Push: source machine sends to remote server](#push-source-machine-sends-to-remote-server)
  - [Pull: backup server fetches from clients](#pull-backup-server-fetches-from-clients)
  - [Push vs Pull — when to use each](#push-vs-pull--when-to-use-each)
- [6. SSH Key Setup for Automated Backups](#6-ssh-key-setup-for-automated-backups)
  - [Restrict backup key to rsync only (security hardening)](#restrict-backup-key-to-rsync-only-security-hardening)
- [7. Snapshot-Style Incrementals with `--link-dest`](#7-snapshot-style-incrementals-with---link-dest)
  - [How it works](#how-it-works)
  - [Basic `--link-dest` usage](#basic---link-dest-usage)
  - [What if yesterday's snapshot doesn't exist?](#what-if-yesterdays-snapshot-doesnt-exist)
- [8. Production Snapshot Backup Script](#8-production-snapshot-backup-script)
  - [Configuration ###](#configuration-)
- [9. Bandwidth Limiting](#9-bandwidth-limiting)
- [10. Partial Transfers and Resume](#10-partial-transfers-and-resume)
- [11. rsync Daemon Mode (Alternative to SSH)](#11-rsync-daemon-mode-alternative-to-ssh)
- [12. Verifying rsync Backups](#12-verifying-rsync-backups)
- [13. Restoring from rsync Snapshots](#13-restoring-from-rsync-snapshots)
- [Lab Exercises](#lab-exercises)
  - [Lab 03-1: Basic local rsync](#lab-03-1-basic-local-rsync)
  - [Lab 03-2: Test --delete behaviour](#lab-03-2-test---delete-behaviour)
  - [Lab 03-3: Snapshot incrementals with --link-dest](#lab-03-3-snapshot-incrementals-with---link-dest)
  - [Lab 03-4: SSH backup to remote (requires two machines)](#lab-03-4-ssh-backup-to-remote-requires-two-machines)
  - [Lab 03-5: Run the production snapshot script](#lab-03-5-run-the-production-snapshot-script)
- [Review Questions](#review-questions)
- [Answers to Review Questions](#answers-to-review-questions)

---

## 1. What is rsync?

`rsync` is a fast, versatile file synchronisation tool. It uses a **delta transfer algorithm** — it only transfers the parts of files that have changed. This makes it extremely efficient for regular backups over slow or expensive links.

Key properties:
- Transfers only changed data (delta algorithm)
- Preserves all file metadata: permissions, ownership, timestamps, symlinks, hardlinks, ACLs, xattrs
- Works locally or over SSH (and other transports)
- Supports include/exclude filters
- Can create efficient incremental backups using hardlinks (`--link-dest`)

```bash
# Install rsync (should already be present on RHEL 10)
sudo dnf install -y rsync

# Verify
rsync --version
```

---

## 2. rsync Flag Reference

### Essential flags

| Flag | Description |
|------|-------------|
| `-a` / `--archive` | Archive mode: equivalent to `-rlptgoD` (see below) |
| `-r` / `--recursive` | Recurse into directories |
| `-l` / `--links` | Copy symlinks as symlinks |
| `-p` / `--perms` | Preserve permissions |
| `-t` / `--times` | Preserve modification times |
| `-g` / `--group` | Preserve group ownership |
| `-o` / `--owner` | Preserve file owner (root only) |
| `-D` | Preserve device files and special files |
| `-v` / `--verbose` | Verbose output |
| `-z` / `--compress` | Compress data during transfer (good for slow links) |
| `-h` / `--human-readable` | Output in human-readable format |

### Sync behaviour flags

| Flag | Description |
|------|-------------|
| `--delete` | Delete destination files not present in source |
| `--delete-before` | Delete before transfer (saves space during sync) |
| `--delete-after` | Delete after transfer (safer, keeps files during transfer) |
| `--delete-delay` | Stage deletions, apply at end |
| `--ignore-existing` | Skip files already in destination |
| `--update` | Skip files newer in destination |
| `--checksum` | Compare by checksum, not size+mtime (slower, thorough) |
| `--size-only` | Compare by size only (fastest, less accurate) |

### Metadata and extended attributes

| Flag | Description |
|------|-------------|
| `-A` / `--acls` | Preserve ACLs (requires `--perms`) |
| `-X` / `--xattrs` | Preserve extended attributes (SELinux labels) |
| `-H` / `--hard-links` | Preserve hard links |
| `--numeric-ids` | Use numeric UID/GID instead of names |
| `--sparse` | Handle sparse files efficiently |

### Backup and incremental flags

| Flag | Description |
|------|-------------|
| `--link-dest=DIR` | Hardlink to DIR for unchanged files (snapshot backups) |
| `-b` / `--backup` | Make backups of files before overwriting |
| `--backup-dir=DIR` | Store backups of overwritten files in DIR |
| `--suffix=SUFFIX` | Append suffix to backup files (default: `~`) |

### Transfer and safety flags

| Flag | Description |
|------|-------------|
| `-n` / `--dry-run` | Show what would be transferred without doing it |
| `--progress` | Show per-file transfer progress |
| `--stats` | Print file transfer statistics at end |
| `--partial` | Keep partial files (resume interrupted transfers) |
| `--bwlimit=RATE` | Limit bandwidth (e.g., `--bwlimit=10m` = 10 MB/s) |
| `--timeout=SECONDS` | Set I/O timeout |

### Path and filter flags

| Flag | Description |
|------|-------------|
| `--exclude=PATTERN` | Exclude files matching pattern |
| `--exclude-from=FILE` | Read exclusions from file |
| `--include=PATTERN` | Include files matching pattern (overrides exclude) |
| `--filter=RULE` | Fine-grained filter rules |
| `-e` / `--rsh=COMMAND` | Specify remote shell (usually `ssh`) |

---

## 3. Trailing Slash Gotcha

The most common rsync mistake. Pay close attention:

```bash
# WITHOUT trailing slash on source: copies the DIRECTORY ITSELF
rsync -av /etc /backup/
# Result: /backup/etc/passwd, /backup/etc/hostname ...

# WITH trailing slash on source: copies the CONTENTS of the directory
rsync -av /etc/ /backup/etc/
# Result: /backup/etc/passwd, /backup/etc/hostname ...
```

**Rule:** 
- `rsync SRC DEST` — copies `SRC` directory into `DEST` → `DEST/SRC/`  
- `rsync SRC/ DEST` — copies *contents* of `SRC` into `DEST` → `DEST/`

Use the trailing slash when you want to keep the same structure. Without it when you want to recreate the directory itself under the destination.

---

## 4. Local Backups

### Simple local sync

```bash
# Sync /etc to /backup/etc (mirror)
sudo rsync -aAXv --delete --numeric-ids /etc/ /backup/etc/

# Dry run first (see what would change)
sudo rsync -aAXvn --delete --numeric-ids /etc/ /backup/etc/
```

### Local backup with stats

```bash
sudo rsync -aAXvh --delete --numeric-ids --stats /etc/ /backup/etc/
```

---

## 5. Remote Backups over SSH

### Push: source machine sends to remote server

```bash
# Push /etc to backup-server
sudo rsync -aAXvz --delete --numeric-ids \
  /etc/ \
  root@backup-server:/backup/clients/$(hostname)/etc/

# With explicit SSH options
sudo rsync -aAXvz --delete --numeric-ids \
  -e "ssh -p 22 -i /root/.ssh/backup_key" \
  /etc/ \
  backupuser@backup-server:/backup/clients/$(hostname)/etc/
```

### Pull: backup server fetches from clients

```bash
# On backup-server: pull /etc from backup-client
sudo rsync -aAXvz --delete --numeric-ids \
  -e "ssh -p 22 -i /root/.ssh/backup_key" \
  root@backup-client:/etc/ \
  /backup/clients/backup-client/etc/
```

### Push vs Pull — when to use each

| Approach | Pros | Cons |
|----------|------|------|
| **Push (client → server)** | Simple, client controls its own backup | Client needs write access to backup server |
| **Pull (server → client)** | Centralized control, client cannot delete backups | Server needs read access to all clients |

**Security recommendation:** Pull model. The backup server has read-only access to clients. A compromised client cannot delete its own backups.

---

## 6. SSH Key Setup for Automated Backups

Automated backups require passwordless SSH authentication.

```bash
# On the backup server: generate a dedicated backup key
sudo ssh-keygen -t ed25519 -f /root/.ssh/backup_key -N "" -C "backup-automation"

# Display the public key
sudo cat /root/.ssh/backup_key.pub

# On each client: add the public key to authorized_keys
# (run on client, or use ssh-copy-id from server)
sudo mkdir -p /root/.ssh
sudo chmod 700 /root/.ssh

# On backup-server, push the key to client
sudo ssh-copy-id -i /root/.ssh/backup_key.pub root@backup-client

# Test passwordless connection
sudo ssh -i /root/.ssh/backup_key root@backup-client hostname
```

### Restrict backup key to rsync only (security hardening)

In `/root/.ssh/authorized_keys` on the client, prefix the key with restrictions:

```
command="rsync --server --daemon .",no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty ssh-ed25519 AAAA... backup-automation
```

Or use `rrsync` (restricted rsync) which limits the key to a specific directory:

```bash
# Install rrsync if available, or copy from rsync source
sudo cp /usr/share/doc/rsync/support/rrsync /usr/local/bin/rrsync
sudo chmod +x /usr/local/bin/rrsync

# In authorized_keys on the client:
# command="/usr/local/bin/rrsync -ro /",no-port-forwarding,...
```

---

## 7. Snapshot-Style Incrementals with `--link-dest`

The `--link-dest` option is one of the most powerful rsync features. It enables **space-efficient incremental backups** where unchanged files are represented as hardlinks to a previous backup rather than new copies.

### How it works

1. First run: full copy of source → `DEST/2026-02-17/`
2. Second run: copy changed files to `DEST/2026-02-18/`, hardlink unchanged files to `DEST/2026-02-17/`
3. Third run: copy changed files to `DEST/2026-02-19/`, hardlink unchanged files to `DEST/2026-02-18/`

Each day's directory appears to contain a complete copy of the source, but unchanged files are just hardlinks — they use virtually no additional disk space.

```
/backup/
├── 2026-02-17/   ← Full copy (real files)
│   └── etc/
│       ├── passwd       [real file, 2 KB]
│       └── hostname     [real file, 1 KB]
├── 2026-02-18/   ← Changed files are new; unchanged are hardlinks
│   └── etc/
│       ├── passwd       [hardlink → 2026-02-17/etc/passwd]  (0 extra space)
│       └── hostname     [NEW FILE — hostname changed]       (1 KB)
└── 2026-02-19/   ← Same principle
    └── etc/
        ...
```

**Disk space used:** Only the unique data across all snapshots. Restoring any single day requires only that day's directory — no archive chain needed.

### Basic `--link-dest` usage

```bash
DEST=/backup/snapshots
TODAY=$(date +%Y-%m-%d)
YESTERDAY=$(date -d "yesterday" +%Y-%m-%d)

# Create today's snapshot, hardlinking to yesterday
sudo rsync -aAXv --delete --numeric-ids \
  --link-dest=${DEST}/${YESTERDAY} \
  /etc/ \
  ${DEST}/${TODAY}/etc/
```

### What if yesterday's snapshot doesn't exist?

`rsync` performs a full copy automatically when `--link-dest` points to a non-existent directory. This makes the script safe to run even on the first run.

---

## 8. Production Snapshot Backup Script

```bash
sudo tee /usr/local/bin/rsync-snapshot.sh <<'SCRIPT'
#!/bin/bash
# rsync-snapshot.sh — Daily snapshot backup with --link-dest
# Creates space-efficient daily snapshots, retains last 30 days

set -euo pipefail

### Configuration ###
BACKUP_ROOT="/backup/snapshots"
REMOTE_HOST=""                        # Set to "user@host" for remote, leave empty for local
REMOTE_PORT="22"
SSH_KEY="/root/.ssh/backup_key"
LOG_FILE="/var/log/rsync-backup.log"
RETENTION_DAYS=30
HOST=$(hostname -s)
TODAY=$(date +%Y-%m-%d)
YESTERDAY=$(date -d "yesterday" +%Y-%m-%d)

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
  "--exclude=*.swp"
  "--exclude=*.tmp"
)

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "${LOG_FILE}"; }

# SSH options for remote
if [[ -n "${REMOTE_HOST}" ]]; then
  SSH_OPTS="-e ssh -p ${REMOTE_PORT} -i ${SSH_KEY} -o StrictHostKeyChecking=no"
  DEST_BASE="${REMOTE_HOST}:${BACKUP_ROOT}/${HOST}"
  # Create remote directories
  ssh -p "${REMOTE_PORT}" -i "${SSH_KEY}" "${REMOTE_HOST}" \
    "mkdir -p ${BACKUP_ROOT}/${HOST}/${TODAY}"
else
  SSH_OPTS=""
  DEST_BASE="${BACKUP_ROOT}/${HOST}"
  mkdir -p "${DEST_BASE}/${TODAY}"
fi

log "=== rsync snapshot START: ${HOST} → ${TODAY} ==="

for SRC in "${SOURCES[@]}"; do
  [ -d "${SRC}" ] || { log "SKIP: ${SRC} not found"; continue; }

  SRCNAME=$(echo "${SRC}" | tr '/' '-' | sed 's/^-//')
  DEST="${DEST_BASE}/${TODAY}/${SRCNAME}"
  LINKDEST="${DEST_BASE}/${YESTERDAY}/${SRCNAME}"

  log "Syncing ${SRC} → ${DEST}"

  rsync -aAXhz --delete --numeric-ids \
    ${SSH_OPTS} \
    ${EXCLUDES[@]} \
    --link-dest="${LINKDEST}" \
    "${SRC}/" \
    "${DEST}/" \
    2>>"${LOG_FILE}" \
    && log "OK: ${SRC}" \
    || log "ERROR: ${SRC} failed (exit $?)"
done

# Cleanup old snapshots
log "Cleaning snapshots older than ${RETENTION_DAYS} days"
if [[ -z "${REMOTE_HOST}" ]]; then
  find "${DEST_BASE}" -maxdepth 1 -type d -name "????-??-??" \
    | sort | head -n -${RETENTION_DAYS} \
    | xargs -r rm -rf
fi

log "=== rsync snapshot END ==="
SCRIPT

sudo chmod +x /usr/local/bin/rsync-snapshot.sh
```

---

## 9. Bandwidth Limiting

For backups over metered or congested links:

```bash
# Limit to 50 MB/s
rsync -aAXvz --delete --bwlimit=50m /etc/ backup-server:/backup/etc/

# For very slow links (2 MB/s)
rsync -aAXvz --delete --bwlimit=2m \
  -e "ssh -C"  \          # SSH compression
  /etc/ backup-server:/backup/etc/
```

---

## 10. Partial Transfers and Resume

For large backups over unreliable connections:

```bash
# Enable partial transfers (file is kept if transfer interrupted)
rsync -aAXvz --delete --partial --progress \
  /var/lib/mysql/ backup-server:/backup/mysql/

# Use --partial-dir to store partials in a staging dir
rsync -aAXvz --delete --partial-dir=.rsync-partial \
  /var/lib/mysql/ backup-server:/backup/mysql/
```

---

## 11. rsync Daemon Mode (Alternative to SSH)

For very high-performance local network backups, rsync daemon mode is faster than SSH (no encryption overhead):

```bash
# On backup server: configure rsync daemon
sudo tee /etc/rsyncd.conf <<'EOF'
# /etc/rsyncd.conf
uid = root
gid = root
use chroot = yes
max connections = 4
log file = /var/log/rsyncd.log
pid file = /var/run/rsyncd.pid
lock file = /var/run/rsync.lock

[backups]
  path = /backup
  comment = Backup storage
  read only = no
  list = no
  auth users = backupuser
  secrets file = /etc/rsyncd.secrets
EOF

# Create secrets file
sudo tee /etc/rsyncd.secrets <<'EOF'
backupuser:strongpassword123
EOF
sudo chmod 600 /etc/rsyncd.secrets

# Enable and start rsync daemon
sudo systemctl enable --now rsyncd

# Open firewall port
sudo firewall-cmd --permanent --add-service=rsyncd
sudo firewall-cmd --reload
```

```bash
# Client: connect to rsync daemon (no SSH)
rsync -aAXv --delete \
  /etc/ backupuser@backup-server::backups/$(hostname)/etc/
```

**Note:** Rsync daemon mode transmits passwords in cleartext unless combined with SSH tunnel or used only on trusted internal networks.

---

## 12. Verifying rsync Backups

```bash
# Dry-run compare: show differences between source and backup
sudo rsync -aAXvn --delete --checksum --numeric-ids \
  /etc/ /backup/snapshots/$(hostname)/$(date +%Y-%m-%d)/etc/

# Count files in backup vs source
echo "Source files:"; find /etc -type f | wc -l
echo "Backup files:"; find /backup/snapshots/$(hostname)/$(date +%Y-%m-%d)/etc -type f | wc -l

# Deep checksum comparison (slow but thorough)
sudo rsync -aAXvn --checksum /etc/ /backup/snapshots/$(hostname)/$(date +%Y-%m-%d)/etc/
# Any output means those files differ
```

---

## 13. Restoring from rsync Snapshots

Since each snapshot directory is a complete copy:

```bash
# Restore entire /etc from yesterday's snapshot
sudo rsync -aAXv --delete --numeric-ids \
  /backup/snapshots/$(hostname)/$(date -d "yesterday" +%Y-%m-%d)/etc/ \
  /etc/

# Restore a single file
sudo cp /backup/snapshots/$(hostname)/2026-02-17/etc/ssh/sshd_config \
        /etc/ssh/sshd_config

# Restore a specific date's /home to an alternate location
sudo rsync -aAXv --numeric-ids \
  /backup/snapshots/$(hostname)/2026-02-10/home/ \
  /restore/home-20260210/
```

---

## Lab Exercises

### Lab 03-1: Basic local rsync

```bash
# 1. Create source data
mkdir -p /tmp/rsync_src/{docs,configs}
cp /etc/hostname /tmp/rsync_src/configs/
echo "hello world" > /tmp/rsync_src/docs/test.txt

# 2. Sync to destination
mkdir -p /tmp/rsync_dst
rsync -aAXvh /tmp/rsync_src/ /tmp/rsync_dst/

# 3. Verify
ls -la /tmp/rsync_dst/

# 4. Make a change and sync again (observe delta transfer)
echo "new content" >> /tmp/rsync_src/docs/test.txt
rsync -aAXvh /tmp/rsync_src/ /tmp/rsync_dst/
# Only the changed file should show in output
```

### Lab 03-2: Test --delete behaviour

```bash
# 1. Add a file to destination that isn't in source
touch /tmp/rsync_dst/should_be_deleted.txt

# 2. Dry run with --delete to see what would happen
rsync -aAXvhn --delete /tmp/rsync_src/ /tmp/rsync_dst/
# "deleting should_be_deleted.txt" should appear

# 3. Actually sync with --delete
rsync -aAXvh --delete /tmp/rsync_src/ /tmp/rsync_dst/
ls /tmp/rsync_dst/  # File should be gone
```

### Lab 03-3: Snapshot incrementals with --link-dest

```bash
# 1. First snapshot (acts as full)
mkdir -p /backup/lab_snapshots/{2026-02-17,2026-02-18}
rsync -aAXvh --delete --numeric-ids \
  /tmp/rsync_src/ \
  /backup/lab_snapshots/2026-02-17/

# 2. Modify source
echo "day 2 content" > /tmp/rsync_src/docs/day2.txt

# 3. Second snapshot with --link-dest
rsync -aAXvh --delete --numeric-ids \
  --link-dest=/backup/lab_snapshots/2026-02-17 \
  /tmp/rsync_src/ \
  /backup/lab_snapshots/2026-02-18/

# 4. Verify hardlinks (unchanged files)
ls -li /backup/lab_snapshots/2026-02-17/configs/hostname
ls -li /backup/lab_snapshots/2026-02-18/configs/hostname
# Inode numbers should be IDENTICAL — they are hardlinks

# 5. Check disk usage
du -sh /backup/lab_snapshots/2026-02-17/
du -sh /backup/lab_snapshots/2026-02-18/
# Second snapshot uses much less space despite appearing complete
```

### Lab 03-4: SSH backup to remote (requires two machines)

```bash
# On backup-server (receiver):
sudo mkdir -p /backup/clients/$(hostname -s from client)

# On backup-client (sender):
# 1. Generate SSH key
sudo ssh-keygen -t ed25519 -f /root/.ssh/backup_key -N ""

# 2. Copy key to server (enter password when prompted)
sudo ssh-copy-id -i /root/.ssh/backup_key.pub root@backup-server

# 3. Test connection
sudo ssh -i /root/.ssh/backup_key root@backup-server hostname

# 4. Perform remote rsync
sudo rsync -aAXvz --delete --numeric-ids \
  -e "ssh -i /root/.ssh/backup_key" \
  /etc/ \
  root@backup-server:/backup/clients/$(hostname -s)/etc/
```

### Lab 03-5: Run the production snapshot script

```bash
# Run the snapshot script
sudo /usr/local/bin/rsync-snapshot.sh

# Review the log
sudo cat /var/log/rsync-backup.log

# List snapshots
ls -la /backup/snapshots/$(hostname -s)/
```

---

## Review Questions

1. What does the `--archive` (`-a`) flag do, and which individual flags does it combine?
2. Explain the trailing slash rule in rsync — what is the difference between `rsync src dest` and `rsync src/ dest`?
3. What does `--link-dest` do, and why does it save disk space?
4. What is the difference between pushing and pulling backups, and which is considered more secure?
5. Why is `--delete` important for backups, and what risk does it carry?
6. What does `--dry-run` do and when should you use it?
7. What three flags preserve SELinux labels, ACLs, and extended attributes in rsync?
8. How do you restore a single file from an rsync snapshot backup?
9. What is the difference between rsync SSH transport and rsync daemon mode?
10. How do you verify that an rsync backup is complete and accurate?

---

## Answers to Review Questions

1. `-a` = `-rlptgoD`: **r**ecursive, **l**inks (symlinks), **p**ermissions, **t**imestamps, **g**roup, **o**wner, **D**evice and special files. It is the standard "preserve everything" combination.
2. `rsync src dest` copies the `src` directory itself into `dest`, resulting in `dest/src/`. `rsync src/ dest` copies the *contents* of `src` into `dest`, resulting in `dest/` containing what was inside `src/`.
3. `--link-dest=PREVDIR` tells rsync to create hardlinks in the destination for files that are identical to those in `PREVDIR`. Hardlinks share the same disk blocks — so unchanged files occupy zero additional space. Each snapshot appears complete but only new/changed data actually uses space.
4. **Push** = client sends data to server. **Pull** = server fetches from client. Pull is more secure because the backup server initiates connections; a compromised client cannot reach into the backup server to modify or delete backups.
5. `--delete` removes files from the destination that no longer exist in the source, keeping the backup a true mirror. Risk: if run incorrectly (wrong paths) it will delete valid backup data. Always test with `--dry-run` first.
6. `--dry-run` (`-n`) shows exactly what rsync would do (transfers, deletions, renames) without actually doing it. Use before any destructive or large sync, especially with `--delete`.
7. `-A` (`--acls`), `-X` (`--xattrs`), `--numeric-ids` for numeric ownership. The SELinux context is stored as an xattr, so `-X` covers SELinux.
8. `cp /backup/snapshots/HOSTNAME/DATE/path/to/file /original/path/to/file` — rsync snapshots are plain directories; any single file can be copied directly using `cp` or `rsync`.
9. **SSH transport** encrypts data in transit using SSH; slower due to encryption overhead but secure. **Daemon mode** uses the rsync protocol directly (port 873) without encryption; faster but only suitable for trusted internal networks.
10. Run rsync again with `--checksum` and `--dry-run` — any output means the files differ. Also compare file counts between source and backup. For deep verification, use `diff -rq`.

---

*Previous: [02 — tar](02-tar.md)*
*Next: [04 — dump / xfsdump](04-dump-restore.md)*
