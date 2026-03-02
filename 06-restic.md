# Module 06 — Restic — Modern Deduplicated Encrypted Backups
[![CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey)](./LICENSE.md)
[![RHEL 10](https://img.shields.io/badge/platform-RHEL%2010-red)](https://access.redhat.com/products/red-hat-enterprise-linux)
[![Backup](https://img.shields.io/badge/Backup-RHEL%2010-blue)](https://access.redhat.com/products/red-hat-enterprise-linux)

## Learning Objectives

By the end of this module you will be able to:

- Install and configure Restic on RHEL 10
- Initialise and use repositories with local, SFTP, AWS S3, and self-hosted MinIO backends
- Run backups with exclusions, tags, and custom options
- List, browse, and restore snapshots
- Configure and apply forget/prune retention policies
- Verify repository integrity
- Automate Restic with systemd timers
- Understand Restic's deduplication, encryption, and repository format

---

## Table of Contents

- [1. What is Restic?](#1-what-is-restic)
  - [Key features](#key-features)
  - [Restic vs tar/rsync](#restic-vs-tarrsync)
- [2. Installation](#2-installation)
  - [2.1 Install from RHEL 10 repositories](#21-install-from-rhel-10-repositories)
  - [2.2 Install latest binary directly](#22-install-latest-binary-directly)
  - [2.3 Enable auto-update (optional)](#23-enable-auto-update-optional)
- [3. Core Concepts](#3-core-concepts)
  - [Repository](#repository)
  - [Snapshot](#snapshot)
  - [Pack / Blob / Tree](#pack--blob--tree)
  - [Password / Key](#password--key)
- [4. Backend A: Local Filesystem](#4-backend-a-local-filesystem)
  - [4.1 Initialise a local repository](#41-initialise-a-local-repository)
  - [4.2 First backup](#42-first-backup)
  - [4.3 Subsequent backups (automatic incremental)](#43-subsequent-backups-automatic-incremental)
- [5. Listing and Browsing Snapshots](#5-listing-and-browsing-snapshots)
- [6. Restoring from Snapshots](#6-restoring-from-snapshots)
  - [6.1 Restore entire snapshot](#61-restore-entire-snapshot)
  - [6.2 Restore specific files or directories](#62-restore-specific-files-or-directories)
  - [6.3 Mount a snapshot as a FUSE filesystem](#63-mount-a-snapshot-as-a-fuse-filesystem)
- [7. Tags](#7-tags)
- [8. Forget and Prune — Retention Policy](#8-forget-and-prune--retention-policy)
  - [8.1 forget — remove snapshots per policy](#81-forget--remove-snapshots-per-policy)
  - [8.2 forget policy options](#82-forget-policy-options)
  - [8.3 prune — remove orphaned data](#83-prune--remove-orphaned-data)
- [9. Check — Repository Integrity Verification](#9-check--repository-integrity-verification)
- [10. Backend B: SFTP](#10-backend-b-sftp)
  - [10.1 Configure SFTP access](#101-configure-sftp-access)
  - [10.2 Initialise SFTP repository](#102-initialise-sftp-repository)
  - [10.3 Backup to SFTP](#103-backup-to-sftp)
  - [10.4 Store SFTP options in a config file](#104-store-sftp-options-in-a-config-file)
- [11. Backend C: AWS S3](#11-backend-c-aws-s3)
  - [11.1 Prerequisites](#111-prerequisites)
  - [11.2 Minimal IAM policy for Restic](#112-minimal-iam-policy-for-restic)
  - [11.3 Configure credentials](#113-configure-credentials)
  - [11.4 Initialise and use S3 repository](#114-initialise-and-use-s3-repository)
  - [11.5 S3-specific options](#115-s3-specific-options)
- [12. Backend D: Self-Hosted MinIO (S3-Compatible)](#12-backend-d-self-hosted-minio-s3-compatible)
  - [12.1 Install MinIO server](#121-install-minio-server)
  - [12.2 Configure MinIO as a systemd service](#122-configure-minio-as-a-systemd-service)
  - [12.3 Create a MinIO bucket](#123-create-a-minio-bucket)
  - [12.4 Create a dedicated MinIO user for backups](#124-create-a-dedicated-minio-user-for-backups)
  - [12.5 Initialise Restic repository on MinIO](#125-initialise-restic-repository-on-minio)
- [13. Exclude Patterns File](#13-exclude-patterns-file)
- [14. Production Restic Backup Script with systemd](#14-production-restic-backup-script-with-systemd)
  - [14.1 The backup script](#141-the-backup-script)
  - [14.2 systemd service and timer](#142-systemd-service-and-timer)
  - [14.3 Multiple repositories with separate timers](#143-multiple-repositories-with-separate-timers)
- [15. Restic Key Management](#15-restic-key-management)
- [Lab Exercises](#lab-exercises)
  - [Lab 06-1: Local repository — full cycle](#lab-06-1-local-repository--full-cycle)
  - [Lab 06-2: Forget and prune](#lab-06-2-forget-and-prune)
  - [Lab 06-3: Check repository integrity](#lab-06-3-check-repository-integrity)
  - [Lab 06-4: MinIO integration (requires MinIO running)](#lab-06-4-minio-integration-requires-minio-running)
  - [Lab 06-5: Set up and trigger the systemd timer](#lab-06-5-set-up-and-trigger-the-systemd-timer)
- [Review Questions](#review-questions)
- [Answers to Review Questions](#answers-to-review-questions)

---

## 1. What is Restic?

Restic is a modern, open-source backup tool written in Go. It has become the standard choice for encrypted, deduplicated, cloud-friendly backups.

### Key features

| Feature | Description |
|---------|-------------|
| **Encryption** | All data encrypted with AES-256-CTR before leaving the machine — always, no option to disable |
| **Deduplication** | Content-defined chunking deduplicates data across all snapshots |
| **Incremental** | Only new/changed chunks are uploaded — automatically |
| **Multiple backends** | Local, SFTP, AWS S3, MinIO, Azure, Google Cloud, Backblaze B2, REST server |
| **Portable** | Single binary, no daemon required |
| **Integrity verification** | Built-in `check` and `check --read-data` commands |
| **Concurrent access** | Multiple clients can safely back up to the same repository |

### Restic vs tar/rsync

| | Restic | tar/rsync |
|--|--------|-----------|
| Encryption | Built-in, mandatory | External tool needed |
| Deduplication | Yes (content-aware) | No (rsync is file-level) |
| Cloud backends | Native | Requires FUSE/s3fs |
| Repository format | Proprietary (but open) | Standard archive |
| Single-file restore | Yes | Yes |
| Resume interrupted backup | Yes | Yes (rsync) |

[↑ Table of Contents](#table-of-contents)

---

## 2. Installation

### 2.1 Install from RHEL 10 repositories

```bash
# Restic is available in EPEL
sudo dnf install -y epel-release
sudo dnf install -y restic

# Verify
restic version
```

### 2.2 Install latest binary directly

If EPEL's version is behind:

```bash
# Download latest binary from GitHub releases
# Check https://github.com/restic/restic/releases for latest version
RESTIC_VERSION="0.17.3"
curl -L "https://github.com/restic/restic/releases/download/v${RESTIC_VERSION}/restic_${RESTIC_VERSION}_linux_amd64.bz2" \
  | bunzip2 > /usr/local/bin/restic

sudo chmod +x /usr/local/bin/restic

# Verify
restic version
```

### 2.3 Enable auto-update (optional)

```bash
# Restic can update itself
sudo restic self-update
```

[↑ Table of Contents](#table-of-contents)

---

## 3. Core Concepts

### Repository

A Restic **repository** is the storage location for all backup data. It can be on a local disk, a remote server, or a cloud object store. All data in the repository is encrypted.

```
Repository structure:
  repo/
  ├── config          # Repository configuration (encrypted)
  ├── data/           # Encrypted data packs (deduplicated chunks)
  ├── index/          # Index of all data packs
  ├── keys/           # Encryption key(s)
  ├── locks/          # Process locks to prevent concurrent writes
  └── snapshots/      # Snapshot metadata (one file per snapshot)
```

### Snapshot

A **snapshot** is a point-in-time record of what was backed up. Each snapshot stores:
- The list of files and their metadata
- References to deduplicated data chunks in the `data/` directory
- Tags, hostname, username, timestamp

### Pack / Blob / Tree

Internally, Restic chunks data into variable-size **blobs**, groups them into **packs**, and stores packs in the `data/` directory. This enables cross-snapshot deduplication.

### Password / Key

The repository is encrypted with a key derived from a password. The password is never stored — you must provide it every time. Multiple keys can be stored in a repository (useful for team access).

[↑ Table of Contents](#table-of-contents)

---

## 4. Backend A: Local Filesystem

### 4.1 Initialise a local repository

```bash
# Choose a strong password and initialise
export RESTIC_REPOSITORY="/backup/restic-repo"
export RESTIC_PASSWORD="your-strong-password-here"

# Initialise the repository
restic init

# Or specify inline
restic -r /backup/restic-repo init
```

**Storing the password:** Never hard-code passwords in scripts. Use:
- `RESTIC_PASSWORD` environment variable
- `RESTIC_PASSWORD_FILE` pointing to a file readable only by root
- `RESTIC_PASSWORD_COMMAND` to read from a secrets manager

```bash
# Create a password file (root-only)
echo "your-strong-password-here" | sudo tee /etc/backup/restic-password
sudo chmod 600 /etc/backup/restic-password
sudo chown root:root /etc/backup/restic-password
```

### 4.2 First backup

```bash
export RESTIC_REPOSITORY="/backup/restic-repo"
export RESTIC_PASSWORD_FILE="/etc/backup/restic-password"

# Back up /etc and /home
sudo -E restic backup /etc /home /root /boot /var/www \
  --exclude /proc \
  --exclude /sys \
  --exclude /dev \
  --exclude /run \
  --exclude /tmp \
  --exclude /var/tmp \
  --exclude /var/cache \
  --exclude /backup

# With verbose output and progress
sudo -E restic backup /etc /home \
  --verbose \
  --exclude /proc --exclude /sys --exclude /dev \
  --exclude /run --exclude /tmp --exclude /var/tmp
```

### 4.3 Subsequent backups (automatic incremental)

Every Restic backup is automatically incremental — it only transfers and stores new/changed chunks:

```bash
# Subsequent run — Restic detects what changed
sudo -E restic backup /etc /home /root
# Output shows: X files changed, Y bytes added, Z bytes unchanged
```

[↑ Table of Contents](#table-of-contents)

---

## 5. Listing and Browsing Snapshots

```bash
# List all snapshots
sudo -E restic snapshots

# Example output:
# ID        Time                 Host        Tags        Paths
# --------  -------------------  ----------  ----------  -----------
# a1b2c3d4  2026-02-17 02:00:05  web01                   /etc /home
# e5f6a7b8  2026-02-18 02:00:07  web01                   /etc /home

# List snapshots for a specific host
sudo -E restic snapshots --host $(hostname)

# List with more detail
sudo -E restic snapshots --verbose

# Show what files are in a specific snapshot
sudo -E restic ls a1b2c3d4

# Show files matching a path in a snapshot
sudo -E restic ls a1b2c3d4 /etc/ssh/
```

[↑ Table of Contents](#table-of-contents)

---

## 6. Restoring from Snapshots

### 6.1 Restore entire snapshot

```bash
# Restore a snapshot to its original paths
sudo -E restic restore a1b2c3d4 --target /

# Restore to alternate location
sudo -E restic restore a1b2c3d4 --target /restore/

# Restore the latest snapshot
sudo -E restic restore latest --target /restore/
```

### 6.2 Restore specific files or directories

```bash
# Restore only /etc/ssh/ from a snapshot
sudo -E restic restore latest \
  --include /etc/ssh/ \
  --target /

# Restore a single file
sudo -E restic restore latest \
  --include /etc/nginx/nginx.conf \
  --target /restore/

# Restore files matching a pattern
sudo -E restic restore latest \
  --include "*.conf" \
  --target /restore/
```

### 6.3 Mount a snapshot as a FUSE filesystem

Restic can mount a snapshot as a browsable filesystem. Requires FUSE.

```bash
# Install FUSE
sudo dnf install -y fuse

# Mount a snapshot
sudo mkdir -p /mnt/restic
sudo -E restic mount /mnt/restic &

# Browse the snapshot
ls /mnt/restic/
ls /mnt/restic/hosts/web01/latest/etc/

# Copy specific files
sudo cp /mnt/restic/hosts/web01/latest/etc/ssh/sshd_config /etc/ssh/sshd_config.restored

# Unmount
sudo umount /mnt/restic
# or: kill %1
```

[↑ Table of Contents](#table-of-contents)

---

## 7. Tags

Tags let you label snapshots for organisation and filtering:

```bash
# Backup with tags
sudo -E restic backup /etc --tag daily --tag production

# List snapshots with a specific tag
sudo -E restic snapshots --tag daily

# Modify tags on an existing snapshot
sudo -E restic tag --add weekly a1b2c3d4
sudo -E restic tag --remove daily a1b2c3d4
sudo -E restic tag --set "monthly,2026" a1b2c3d4
```

[↑ Table of Contents](#table-of-contents)

---

## 8. Forget and Prune — Retention Policy

Restic separates **forgetting** (removing snapshot metadata) from **pruning** (removing actual data).

### 8.1 forget — remove snapshots per policy

```bash
# Keep:
# - last 7 daily snapshots
# - last 4 weekly snapshots
# - last 6 monthly snapshots
# - last 1 yearly snapshot
sudo -E restic forget \
  --keep-daily 7 \
  --keep-weekly 4 \
  --keep-monthly 6 \
  --keep-yearly 1

# Dry run — shows what would be removed without doing it
sudo -E restic forget \
  --keep-daily 7 --keep-weekly 4 --keep-monthly 6 \
  --dry-run

# Forget AND prune in one command (saves time)
sudo -E restic forget \
  --keep-daily 7 \
  --keep-weekly 4 \
  --keep-monthly 6 \
  --keep-yearly 1 \
  --prune
```

### 8.2 forget policy options

| Flag | Description |
|------|-------------|
| `--keep-last N` | Keep the last N snapshots regardless of time |
| `--keep-hourly N` | Keep last N hourly snapshots |
| `--keep-daily N` | Keep last N daily snapshots |
| `--keep-weekly N` | Keep last N weekly snapshots |
| `--keep-monthly N` | Keep last N monthly snapshots |
| `--keep-yearly N` | Keep last N yearly snapshots |
| `--keep-within DURATION` | Keep all snapshots within duration (e.g., `30d`, `1y`) |
| `--keep-tag TAG` | Keep all snapshots with this tag |
| `--group-by GROUP` | Group by host/paths/tags before applying policy |
| `--host HOST` | Only affect snapshots for this host |
| `--dry-run` | Show what would be removed, don't remove |
| `--prune` | Also prune data after forgetting |

### 8.3 prune — remove orphaned data

After `forget` removes snapshot references, `prune` removes the actual data packs that are no longer referenced by any snapshot:

```bash
# Run prune separately (more control)
sudo -E restic prune

# More aggressive: also repack partially used packs
sudo -E restic prune --repack-small
```

[↑ Table of Contents](#table-of-contents)

---

## 9. Check — Repository Integrity Verification

```bash
# Verify repository structure (metadata only — fast)
sudo -E restic check

# Verify ALL data (read every pack — slow but thorough)
sudo -E restic check --read-data

# Verify a random subset of data (good balance)
sudo -E restic check --read-data-subset=10%

# Example output when healthy:
# using temporary cache in /tmp/restic-check...
# load indexes
# check all packs
# check snapshots, trees and blobs
# no errors were found
```

**Recommendation:** Run `restic check` after every forget/prune. Run `restic check --read-data-subset=10%` weekly. Run `restic check --read-data` monthly.

[↑ Table of Contents](#table-of-contents)

---

## 10. Backend B: SFTP

### 10.1 Configure SFTP access

```bash
# Set up SSH key for the backup server (if not already done — see Module 03)
sudo ssh-keygen -t ed25519 -f /root/.ssh/restic_sftp_key -N ""
sudo ssh-copy-id -i /root/.ssh/restic_sftp_key.pub backupuser@backup-server

# Test SFTP connection
sftp -i /root/.ssh/restic_sftp_key backupuser@backup-server
```

### 10.2 Initialise SFTP repository

```bash
# Repository URL format: sftp:user@host:/path
export RESTIC_REPOSITORY="sftp:backupuser@backup-server:/backup/restic/$(hostname)"
export RESTIC_PASSWORD_FILE="/etc/backup/restic-password"

# Pass SSH key via sftp command option
export RESTIC_REPOSITORY="sftp:backupuser@backup-server:/backup/restic/$(hostname)"

# Specify SSH key in repository URL (via sftp options)
sudo -E restic -o sftp.args="-i /root/.ssh/restic_sftp_key" init

# Initialise
sudo -E restic -o sftp.args="-i /root/.ssh/restic_sftp_key -o StrictHostKeyChecking=no" init
```

### 10.3 Backup to SFTP

```bash
sudo -E restic \
  -o sftp.args="-i /root/.ssh/restic_sftp_key -o StrictHostKeyChecking=no" \
  backup /etc /home /root \
  --exclude /proc --exclude /sys --exclude /dev \
  --exclude /run --exclude /tmp --exclude /var/tmp \
  --verbose
```

### 10.4 Store SFTP options in a config file

To avoid repeating `-o sftp.args=...`, use a Restic environment file:

```bash
# /etc/backup/restic-env-sftp
export RESTIC_REPOSITORY="sftp:backupuser@backup-server:/backup/restic/$(hostname)"
export RESTIC_PASSWORD_FILE="/etc/backup/restic-password"
export RESTIC_SFTP_ARGS="-i /root/.ssh/restic_sftp_key -o StrictHostKeyChecking=no"
```

```bash
source /etc/backup/restic-env-sftp
sudo -E restic backup /etc /home
```

[↑ Table of Contents](#table-of-contents)

---

## 11. Backend C: AWS S3

### 11.1 Prerequisites

```bash
# You need:
# - AWS account
# - S3 bucket created (e.g., my-backup-bucket-rhel10)
# - IAM user with S3 access
# - Access Key ID and Secret Access Key
```

### 11.2 Minimal IAM policy for Restic

Create an IAM policy in AWS with these permissions (principle of least privilege):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation",
        "s3:ListBucketMultipartUploads",
        "s3:ListBucketVersions"
      ],
      "Resource": "arn:aws:s3:::my-backup-bucket-rhel10"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:AbortMultipartUpload",
        "s3:ListMultipartUploadParts"
      ],
      "Resource": "arn:aws:s3:::my-backup-bucket-rhel10/*"
    }
  ]
}
```

### 11.3 Configure credentials

```bash
# Store credentials securely
sudo tee /etc/backup/restic-env-s3 <<'EOF'
export RESTIC_REPOSITORY="s3:s3.amazonaws.com/my-backup-bucket-rhel10/$(hostname)"
export RESTIC_PASSWORD_FILE="/etc/backup/restic-password"
export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
export AWS_DEFAULT_REGION="us-east-1"
EOF

sudo chmod 600 /etc/backup/restic-env-s3
```

### 11.4 Initialise and use S3 repository

```bash
source /etc/backup/restic-env-s3

# Initialise
sudo -E restic init

# Backup
sudo -E restic backup /etc /home /root \
  --exclude /proc --exclude /sys --exclude /dev \
  --exclude /run --exclude /tmp --exclude /var/tmp \
  --verbose

# List snapshots
sudo -E restic snapshots
```

### 11.5 S3-specific options

```bash
# Limit concurrent S3 connections (useful for rate limiting)
sudo -E restic -o s3.connections=5 backup /etc

# Use S3-compatible endpoint (for regions that require it)
export RESTIC_REPOSITORY="s3:https://s3.eu-west-2.amazonaws.com/my-bucket/backups"
```

[↑ Table of Contents](#table-of-contents)

---

## 12. Backend D: Self-Hosted MinIO (S3-Compatible)

MinIO is an open-source, self-hosted S3-compatible object store. It runs on RHEL 10 and provides the same S3 API as AWS.

### 12.1 Install MinIO server

```bash
# On the backup server
sudo dnf install -y wget

# Download MinIO server binary
wget https://dl.min.io/server/minio/release/linux-amd64/minio -O /usr/local/bin/minio
sudo chmod +x /usr/local/bin/minio

# Create data directory
sudo mkdir -p /mnt/minio-data
sudo useradd -r -s /sbin/nologin minio-user
sudo chown minio-user:minio-user /mnt/minio-data
```

### 12.2 Configure MinIO as a systemd service

```bash
sudo tee /etc/default/minio <<'EOF'
MINIO_ROOT_USER="minioadmin"
MINIO_ROOT_PASSWORD="miniopassword123"
MINIO_VOLUMES="/mnt/minio-data"
MINIO_OPTS="--console-address :9001"
EOF
sudo chmod 600 /etc/default/minio

sudo tee /etc/systemd/system/minio.service <<'EOF'
[Unit]
Description=MinIO Object Storage
After=network.target
Wants=network.target

[Service]
User=minio-user
Group=minio-user
EnvironmentFile=/etc/default/minio
ExecStart=/usr/local/bin/minio server $MINIO_OPTS $MINIO_VOLUMES
Restart=always
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now minio

# Open firewall ports
sudo firewall-cmd --permanent --add-port=9000/tcp --add-port=9001/tcp
sudo firewall-cmd --reload
```

### 12.3 Create a MinIO bucket

```bash
# Install MinIO client (mc)
wget https://dl.min.io/client/mc/release/linux-amd64/mc -O /usr/local/bin/mc
sudo chmod +x /usr/local/bin/mc

# Configure mc to point at local MinIO
mc alias set local http://backup-server:9000 minioadmin miniopassword123

# Create a bucket for Restic backups
mc mb local/restic-backups

# List buckets
mc ls local/
```

### 12.4 Create a dedicated MinIO user for backups

```bash
# Create a backup-specific user (not the root admin)
mc admin user add local resticuser resticuserpassword

# Create a policy file
cat > /tmp/restic-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:*"],
      "Resource": ["arn:aws:s3:::restic-backups/*", "arn:aws:s3:::restic-backups"]
    }
  ]
}
EOF

# Apply policy
mc admin policy create local restic-policy /tmp/restic-policy.json
mc admin policy attach local restic-policy --user resticuser
```

### 12.5 Initialise Restic repository on MinIO

```bash
sudo tee /etc/backup/restic-env-minio <<'EOF'
export RESTIC_REPOSITORY="s3:http://backup-server:9000/restic-backups/$(hostname)"
export RESTIC_PASSWORD_FILE="/etc/backup/restic-password"
export AWS_ACCESS_KEY_ID="resticuser"
export AWS_SECRET_ACCESS_KEY="resticuserpassword"
EOF
sudo chmod 600 /etc/backup/restic-env-minio

source /etc/backup/restic-env-minio

# Initialise
sudo -E restic init

# Test backup
sudo -E restic backup /etc --verbose

# Verify on MinIO side
mc ls local/restic-backups/$(hostname)/
```

[↑ Table of Contents](#table-of-contents)

---

## 13. Exclude Patterns File

For complex exclusion rules, use `--exclude-file`:

```bash
sudo tee /etc/backup/restic-excludes.txt <<'EOF'
# Restic exclude patterns
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
*.tmp
*.swp
*.pyc
__pycache__
.git
node_modules
EOF

# Use it:
sudo -E restic backup /etc /home --exclude-file=/etc/backup/restic-excludes.txt
```

[↑ Table of Contents](#table-of-contents)

---

## 14. Production Restic Backup Script with systemd

### 14.1 The backup script

```bash
sudo tee /usr/local/bin/restic-backup.sh <<'SCRIPT'
#!/bin/bash
# restic-backup.sh — Daily Restic backup with forget/prune/check

set -euo pipefail

ENV_FILE="${1:-/etc/backup/restic-env-local}"
LOG_FILE="/var/log/restic-backup.log"
EXCLUDE_FILE="/etc/backup/restic-excludes.txt"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "${LOG_FILE}"; }

# Load environment (sets RESTIC_REPOSITORY, RESTIC_PASSWORD_FILE, etc.)
source "${ENV_FILE}"

log "=== Restic backup START: ${RESTIC_REPOSITORY} ==="

# Run backup
log "Backing up..."
restic backup \
  /etc /root /home /boot /var/www /opt /srv /usr/local \
  --exclude-file="${EXCLUDE_FILE}" \
  --tag "daily" \
  --tag "$(date +%A | tr '[:upper:]' '[:lower:]')" \
  --verbose \
  2>>"${LOG_FILE}" && log "Backup OK" || { log "Backup FAILED"; exit 1; }

# Apply retention policy
log "Applying retention policy..."
restic forget \
  --keep-daily 7 \
  --keep-weekly 4 \
  --keep-monthly 6 \
  --keep-yearly 1 \
  --prune \
  2>>"${LOG_FILE}" && log "Forget/prune OK" || log "Forget/prune WARNING"

# Verify integrity
log "Running integrity check..."
restic check \
  --read-data-subset=5% \
  2>>"${LOG_FILE}" && log "Check OK" || log "Check FAILED — investigate!"

log "=== Restic backup END ==="
SCRIPT

sudo chmod +x /usr/local/bin/restic-backup.sh
```

### 14.2 systemd service and timer

```bash
# Create the service unit
sudo tee /etc/systemd/system/restic-backup.service <<'EOF'
[Unit]
Description=Restic Backup
After=network.target
Wants=network.target

[Service]
Type=oneshot
User=root
ExecStart=/usr/local/bin/restic-backup.sh /etc/backup/restic-env-local
StandardOutput=journal
StandardError=journal
SyslogIdentifier=restic-backup
EOF

# Create the timer unit
sudo tee /etc/systemd/system/restic-backup.timer <<'EOF'
[Unit]
Description=Restic Daily Backup Timer

[Timer]
OnCalendar=daily
RandomizedDelaySec=1800
Persistent=true

[Install]
WantedBy=timers.target
EOF

# Enable and start the timer
sudo systemctl daemon-reload
sudo systemctl enable --now restic-backup.timer

# Verify timer is active
sudo systemctl list-timers restic-backup.timer
```

### 14.3 Multiple repositories with separate timers

```bash
# S3 backup — runs weekly at Sunday 03:00
sudo tee /etc/systemd/system/restic-backup-s3.service <<'EOF'
[Unit]
Description=Restic S3 Backup

[Service]
Type=oneshot
User=root
ExecStart=/usr/local/bin/restic-backup.sh /etc/backup/restic-env-s3
EOF

sudo tee /etc/systemd/system/restic-backup-s3.timer <<'EOF'
[Unit]
Description=Restic Weekly S3 Backup Timer

[Timer]
OnCalendar=Sun 03:00:00
Persistent=true

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now restic-backup-s3.timer
```

[↑ Table of Contents](#table-of-contents)

---

## 15. Restic Key Management

A repository can have multiple keys. This allows team members or automation scripts to each have their own password.

```bash
# List keys in the repository
sudo -E restic key list

# Add a new key with a different password
sudo -E restic key add

# Change the current key's password
sudo -E restic key passwd

# Remove a key (use key ID from 'key list')
sudo -E restic key remove <key-id>
```

[↑ Table of Contents](#table-of-contents)

---

## Lab Exercises

### Lab 06-1: Local repository — full cycle

```bash
# 1. Set up environment
export RESTIC_REPOSITORY="/backup/restic-local"
export RESTIC_PASSWORD="labtest123"
mkdir -p /backup/restic-local

# 2. Initialise
restic init

# 3. Back up /etc
sudo -E restic backup /etc --verbose

# 4. Make a change
echo "backup test file" | sudo tee /etc/backup-test.txt

# 5. Back up again
sudo -E restic backup /etc --verbose

# 6. List snapshots and compare
sudo -E restic snapshots
sudo -E restic diff $(restic snapshots --no-lock --json | python3 -c "import sys,json; snaps=json.load(sys.stdin); print(snaps[0]['id'], snaps[1]['id'])")

# 7. Restore the newer file to alternate location
sudo mkdir -p /restore/etc
sudo -E restic restore latest --include /etc/backup-test.txt --target /restore
cat /restore/etc/backup-test.txt
```

### Lab 06-2: Forget and prune

```bash
# Run 5 quick backups to build up snapshots
for i in {1..5}; do
  echo "backup $i at $(date)" | sudo tee /etc/backup-test-$i.txt
  sudo -E restic backup /etc --quiet
done

# List all snapshots
sudo -E restic snapshots

# Apply a retention policy — keep only 3 last
sudo -E restic forget --keep-last 3 --dry-run
sudo -E restic forget --keep-last 3 --prune

# Verify only 3 remain
sudo -E restic snapshots
```

### Lab 06-3: Check repository integrity

```bash
# Quick check
sudo -E restic check

# Read 20% of data
sudo -E restic check --read-data-subset=20%
```

### Lab 06-4: MinIO integration (requires MinIO running)

```bash
# 1. Ensure MinIO is running
systemctl status minio

# 2. Load MinIO environment
source /etc/backup/restic-env-minio

# 3. Initialise Restic repo on MinIO
sudo -E restic init

# 4. Run a backup
sudo -E restic backup /etc /root --verbose

# 5. List snapshots
sudo -E restic snapshots

# 6. Verify from MinIO side
mc ls local/restic-backups/$(hostname)/
```

### Lab 06-5: Set up and trigger the systemd timer

```bash
# Run the service manually to test
sudo systemctl start restic-backup.service
sudo journalctl -u restic-backup.service -n 50

# Check the timer schedule
sudo systemctl list-timers restic-backup.timer

# View the backup log
sudo cat /var/log/restic-backup.log
```

[↑ Table of Contents](#table-of-contents)

---

## Review Questions

1. What encryption algorithm does Restic use, and can it be disabled?
2. What is content-defined chunking and how does it enable deduplication?
3. What is the difference between `restic forget` and `restic prune`?
4. What environment variables are needed to run Restic non-interactively?
5. What command checks repository integrity, and what is the difference between with and without `--read-data`?
6. How do you restore only a specific directory from a Restic snapshot?
7. What is the `--keep-within` flag used for in `restic forget`?
8. How does Restic connect to a MinIO server, and what protocol does it use?
9. What does `restic mount` do, and what package must be installed first?
10. How do you add a second password/key to a Restic repository?

[↑ Table of Contents](#table-of-contents)

---

## Answers to Review Questions

1. Restic uses **AES-256-CTR** for data encryption and **Poly1305-AES** for authentication (authenticated encryption). It **cannot be disabled** — all data is always encrypted. There is no unencrypted repository mode.
2. **Content-defined chunking (CDC)** splits file data into variable-size chunks at content-defined boundaries. The same content in different files (or the same file across snapshots) produces the same chunks with the same hash. These identical chunks are stored only once, regardless of how many snapshots reference them.
3. `restic forget` removes **snapshot metadata** according to a retention policy — it marks old snapshots as deleted but does not free disk space. `restic prune` scans the repository and removes **actual data packs** that are no longer referenced by any remaining snapshot. Both are needed: forget removes the references, prune removes the data.
4. `RESTIC_REPOSITORY` (the repo URL), `RESTIC_PASSWORD` or `RESTIC_PASSWORD_FILE` (the decryption password). For S3/MinIO: also `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`.
5. `restic check`. Without `--read-data`: verifies index integrity and that all referenced packs exist (fast — metadata only). With `--read-data`: downloads and verifies the actual content of all data packs (slow — reads everything). `--read-data-subset=N%` is a good middle ground.
6. `restic restore SNAPSHOT_ID --include /path/to/dir --target /destination/` — restores only files whose path starts with the specified prefix.
7. `--keep-within DURATION` keeps all snapshots taken within a time duration from now. Example: `--keep-within 30d` keeps every snapshot from the last 30 days, regardless of how many there are.
8. Restic connects to MinIO using the **S3 API** (REST over HTTP/HTTPS). MinIO is S3-compatible, so the Restic S3 backend works unchanged. The repository URL format is `s3:http://host:9000/bucket/path`. The credentials are `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`.
9. `restic mount MOUNTPOINT` mounts a virtual filesystem using **FUSE** that exposes all snapshots as browsable directories. The package `fuse` must be installed first. The mount appears as `hosts/HOSTNAME/SNAPSHOTID/` and `hosts/HOSTNAME/latest/`.
10. `restic key add` — you will be prompted for the new password. The new key is stored in the repository's `keys/` directory. Both keys can now be used to access the repository.

[↑ Table of Contents](#table-of-contents)

---

*Previous: [05 — LVM Snapshots](05-lvm-snapshots.md)*
*Next: [07 — Bareos](07-bareos.md)*

© 2026 UncleJS — Licensed under CC BY-NC-SA 4.0
