# Module 05 — LVM Snapshots — Consistent Pre-Backup Freezes
[![CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey)](./LICENSE.md)
[![RHEL 10](https://img.shields.io/badge/platform-RHEL%2010-red)](https://access.redhat.com/products/red-hat-enterprise-linux)
[![Backup](https://img.shields.io/badge/Backup-RHEL%2010-blue)](https://access.redhat.com/products/red-hat-enterprise-linux)

## Learning Objectives

By the end of this module you will be able to:

- Explain how LVM snapshots work at the block level
- Create thick and thin LVM snapshots
- Mount a snapshot read-only for consistent backup
- Use snapshots as a pre-backup consistency mechanism for any tool
- Remove snapshots safely after backup completes
- Monitor snapshot space usage and avoid snapshot overflow
- Script a complete snapshot-based backup workflow

---

## Table of Contents

- [1. Why LVM Snapshots for Backup?](#1-why-lvm-snapshots-for-backup)
- [2. How LVM Snapshots Work (Copy-on-Write)](#2-how-lvm-snapshots-work-copy-on-write)
- [3. Prerequisites](#3-prerequisites)
  - [LVM must be in use](#lvm-must-be-in-use)
  - [Available free space in the Volume Group](#available-free-space-in-the-volume-group)
- [4. Thick Snapshots](#4-thick-snapshots)
  - [4.1 Creating a thick snapshot](#41-creating-a-thick-snapshot)
  - [4.2 Viewing snapshot information](#42-viewing-snapshot-information)
  - [4.3 Mounting a snapshot read-only](#43-mounting-a-snapshot-read-only)
  - [4.4 Removing a snapshot](#44-removing-a-snapshot)
- [5. Thin Snapshots](#5-thin-snapshots)
  - [5.1 When to use thin vs thick snapshots](#51-when-to-use-thin-vs-thick-snapshots)
  - [5.2 Creating a thin pool](#52-creating-a-thin-pool)
  - [5.3 Creating a thin LV and thin snapshot](#53-creating-a-thin-lv-and-thin-snapshot)
  - [5.4 Monitoring thin pool usage](#54-monitoring-thin-pool-usage)
- [6. Snapshot Overflow — The Critical Risk](#6-snapshot-overflow--the-critical-risk)
  - [Preventing overflow](#preventing-overflow)
  - [How much space to allocate?](#how-much-space-to-allocate)
  - [Extending a thick snapshot that is filling up](#extending-a-thick-snapshot-that-is-filling-up)
- [7. Complete Snapshot-Backed Backup Workflow](#7-complete-snapshot-backed-backup-workflow)
  - [7.1 Snapshot + xfsdump](#71-snapshot--xfsdump)
  - [7.2 Snapshot + rsync](#72-snapshot--rsync)
  - [7.3 Snapshot + tar](#73-snapshot--tar)
- [8. Production Snapshot Backup Script](#8-production-snapshot-backup-script)
- [9. Database-Consistent Snapshots](#9-database-consistent-snapshots)
  - [MariaDB/MySQL](#mariadbmysql)
  - [Better approach: use mariabackup](#better-approach-use-mariabackup)
  - [PostgreSQL](#postgresql)
- [10. Restoring from an LVM Snapshot (Direct Method)](#10-restoring-from-an-lvm-snapshot-direct-method)
- [Lab Exercises](#lab-exercises)
  - [Lab 05-1: Create and inspect an LVM snapshot](#lab-05-1-create-and-inspect-an-lvm-snapshot)
  - [Lab 05-2: Mount snapshot and verify consistency](#lab-05-2-mount-snapshot-and-verify-consistency)
  - [Lab 05-3: Back up from snapshot using rsync](#lab-05-3-back-up-from-snapshot-using-rsync)
  - [Lab 05-4: Monitor and clean up](#lab-05-4-monitor-and-clean-up)
  - [Lab 05-5: Run the production snapshot backup script](#lab-05-5-run-the-production-snapshot-backup-script)
- [Review Questions](#review-questions)
- [Answers to Review Questions](#answers-to-review-questions)

---

## 1. Why LVM Snapshots for Backup?

The core problem with backing up a live system is **consistency** — files change while the backup runs. A database writing a transaction, a log file being rotated, a config file being edited — all can produce an inconsistent backup.

LVM snapshots solve this by creating an **instant point-in-time copy** of a logical volume at the block level. The snapshot captures the exact state of the filesystem at one moment, and you back up from that frozen copy rather than the live filesystem.

```
Timeline:
  08:00:00  LVM snapshot created  ← instant, sub-second
  08:00:00  xfsdump/rsync/tar begins reading from snapshot
  08:00:00–08:30:00  Live system continues running, writes normally
  08:30:00  Backup complete
  08:30:01  Snapshot removed
```

The backup sees a consistent state as of 08:00:00. The live system never pauses.

> **Crash-consistent vs application-consistent:** an LVM snapshot on its own is **crash-consistent** — it captures the blocks exactly as they were, including any in-flight database transaction, just as a power failure would. The filesystem journal makes the *filesystem* safe, but applications (especially databases) may still need recovery on restore. For **application-consistent** snapshots, quiesce the application first — see [Section 9](#9-database-consistent-snapshots).

[↑ Table of Contents](#table-of-contents)

---

## 2. How LVM Snapshots Work (Copy-on-Write)

An LVM snapshot does **not** immediately copy all data. Instead, it uses a **Copy-on-Write (CoW)** mechanism:

1. When the snapshot is created, LVM records the current state by noting which blocks exist.
2. The snapshot is initially empty — it contains no data.
3. When the **original volume** is about to overwrite a block, LVM first **copies the original block to the snapshot**, then writes the new data to the original.
4. The snapshot thus always reflects the state of the volume at the moment it was created.

```
Snapshot creation:    [OrigLV: block A=1, block B=2, block C=3]
                      [SnapLV: empty]

Write to OrigLV:      Block A changes from 1 to 5
                      Before write: LVM copies A=1 to SnapLV
                      [OrigLV: block A=5, block B=2, block C=3]
                      [SnapLV: block A=1]

Mount SnapLV:         Reads A=1, B=2, C=3  ← original state
```

**Key implication:** The snapshot occupies only the space of blocks that have changed since the snapshot was taken. For a backup that runs for 30 minutes, you only need enough snapshot space to hold the blocks changed in that 30-minute window.

[↑ Table of Contents](#table-of-contents)

---

## 3. Prerequisites

### LVM must be in use

LVM snapshots require your filesystem to be on an LVM logical volume. This is the default for RHEL 10 installations.

```bash
# Verify LVM is used
pvs    # Physical volumes
vgs    # Volume groups
lvs    # Logical volumes

# Course lab environment (from the README setup):
# VG: backupvg (on /dev/sdb, 20G)
# LVs: datalv (10G, mounted /data) and backuplv (8G, mounted /backup)
#
# A default RHEL 10 install additionally has:
# VG: rhel — LVs: rhel-root (/) and rhel-swap
```

This module's examples and labs snapshot **`backupvg/datalv`** (the `/data` volume from the shared lab environment). Snapshotting a system LV such as `/dev/rhel/root` works identically — it just needs free space in *that* VG, which default installs often lack.

### Available free space in the Volume Group

The snapshot needs free space in the same VG as the source LV.

```bash
# Check free space in each VG
vgs --units g

# Example output (lab environment — 10G + 8G allocated of 20G):
#   VG        #PV #LV #SN Attr   VSize  VFree
#   backupvg    1   2   0 wz--n- 20.00g  2.00g
#   rhel        1   2   0 wz--n- 99.00g     0g
```

You need at least **10–20% of the source LV size** as free space for the snapshot — more if the volume is write-heavy. The lab VG's ~2G free is enough for the 1G snapshots used in this module.

[↑ Table of Contents](#table-of-contents)

---

## 4. Thick Snapshots

Thick snapshots are the traditional LVM snapshot type. They reserve space up front from the VG's free extents.

### 4.1 Creating a thick snapshot

```bash
# Syntax: lvcreate -L SIZE -s -n SNAPNAME SOURCE_LV

# Snapshot the lab data volume (/data)
sudo lvcreate -L 1G -s -n data_snap /dev/backupvg/datalv

# System-VG example: snapshot the root LV
# (works the same — requires free space in the 'rhel' VG)
sudo lvcreate -L 5G -s -n root_snap /dev/rhel/root

# Verify snapshot was created
sudo lvs
# In the Attr column ("swi-a-s---"): position 1 's' = snapshot volume type,
# position 2 'w' = writable, position 3 'i' = inherited allocation policy
```

### 4.2 Viewing snapshot information

```bash
# Detailed snapshot info
sudo lvs -o +snap_percent,origin,lv_snapshot_invalid

# Fields:
# snap_percent  — how full the snapshot CoW space is (CRITICAL — if 100%, snapshot is invalid)
# origin        — the source LV
# lv_snapshot_invalid — 1 if snapshot has overflowed and is no longer valid
```

### 4.3 Mounting a snapshot read-only

```bash
# Create a mount point
sudo mkdir -p /mnt/snapshot

# Mount the snapshot read-only.
# For XFS (the RHEL 10 default) you MUST add 'nouuid' — the snapshot has the
# same filesystem UUID as the still-mounted origin, and a plain mount fails
# with a duplicate-UUID error:
sudo mount -o ro,nouuid /dev/backupvg/data_snap /mnt/snapshot

# Verify
df -h /mnt/snapshot
ls /mnt/snapshot/configs/
```

**Why `nouuid`?** XFS embeds a UUID in the filesystem. The snapshot has the same UUID as the original. Linux will refuse to mount two XFS filesystems with the same UUID without the `nouuid` option.

### 4.4 Removing a snapshot

```bash
# Unmount first
sudo umount /mnt/snapshot

# Remove the snapshot LV
sudo lvremove -f /dev/backupvg/data_snap

# Verify it's gone
sudo lvs
```

[↑ Table of Contents](#table-of-contents)

---

## 5. Thin Snapshots

Thin provisioning uses a **thin pool** — a pool of storage shared among many thin LVs. Thin snapshots are more space-efficient and do not require pre-allocated CoW space.

### 5.1 When to use thin vs thick snapshots

| | Thick Snapshot | Thin Snapshot |
|---|---|---|
| Space reservation | Pre-allocated | Dynamic (from thin pool) |
| Setup complexity | Simple | Requires thin pool |
| Multiple snapshots | Needs space for each | Shares pool efficiently |
| Performance overhead | CoW per write | CoW per write (similar) |
| Available on RHEL 10 | Yes (default LVM) | Yes (with thin pool) |

### 5.2 Creating a thin pool

```bash
# Create a thin pool LV in the lab VG — sized to the ~2G of free space
# (remove any thick snapshots from Section 4 first: lvremove backupvg/data_snap)
sudo lvcreate -L 1G --thinpool thinpool backupvg

# Verify
sudo lvs
# thinpool shows as "twi---tz--"
```

### 5.3 Creating a thin LV and thin snapshot

```bash
# Create a thin LV in the pool. The VIRTUAL size (-V) may exceed the pool —
# thin provisioning over-provisions; blocks are allocated only when written
sudo lvcreate -V 5G --thin -n thindata backupvg/thinpool

# Format and mount the thin LV
sudo mkfs.xfs /dev/backupvg/thindata
sudo mkdir -p /mnt/thindata
sudo mount /dev/backupvg/thindata /mnt/thindata

# Create a thin snapshot (no size required — uses pool space dynamically)
sudo lvcreate -s --name thindata_snap backupvg/thindata

# Mount the thin snapshot
sudo mount -o ro,nouuid /dev/backupvg/thindata_snap /mnt/snapshot
```

> **⚠️ Over-provisioning cuts both ways:** the 5G virtual LV above lives in a 1G pool. If real writes exceed the pool's capacity, **all** thin LVs in the pool error out. Always configure pool auto-extension (5.4) or monitor pool usage closely.

### 5.4 Monitoring thin pool usage

```bash
# Check thin pool data and metadata usage
sudo lvs -a backupvg/thinpool

# Set automatic extension of thin pool (prevent overflow)
sudo lvmconfig activation/thin_pool_autoextend_threshold
# Default: 100 (no auto-extend); set to 80 to auto-extend at 80% full

# In /etc/lvm/lvm.conf (or /etc/lvm/lvmlocal.conf):
sudo tee -a /etc/lvm/lvmlocal.conf <<'EOF'
activation {
    thin_pool_autoextend_threshold = 80
    thin_pool_autoextend_percent = 20
}
EOF
```

[↑ Table of Contents](#table-of-contents)

---

## 6. Snapshot Overflow — The Critical Risk

**If a thick snapshot's CoW space fills up, the snapshot becomes invalid.** A full snapshot cannot be used for backup and must be removed.

### Preventing overflow

```bash
# Monitor snapshot usage (run this frequently during backup)
sudo lvs -o lv_name,snap_percent --noheadings | awk 'NF>1'

# Alert if snapshot is > 70% full
sudo lvs -o lv_name,snap_percent --noheadings | awk '$2 > 70 {print "WARNING: "$1" is "$2"% full"}'
```

### How much space to allocate?

A general rule: allocate **15–20% of the source LV size** for the snapshot, or estimate based on your write rate:

```bash
# How many writes happen per minute on the LV?
# Check iostat for the device
iostat -x 1 10 | grep $(lvs --noheadings -o devices /dev/backupvg/datalv | tr -d ' ')

# Conservative approach: use thin snapshots so overflow is automatically managed
```

### Extending a thick snapshot that is filling up

```bash
# Extend the snapshot CoW space (add 1GB more)
sudo lvextend -L +1G /dev/backupvg/data_snap

# Or resize to a specific size
sudo lvextend -L 2G /dev/backupvg/data_snap
```

### Automatic extension (recommended safety net)

LVM can auto-extend thick snapshots via `dmeventd`. In `/etc/lvm/lvm.conf` (`activation` section):

```
# When a snapshot exceeds 70% usage, grow it by 20% — requires VG free space
snapshot_autoextend_threshold = 70
snapshot_autoextend_percent = 20
```

A threshold of `100` (the default) disables auto-extension. This is the standard mitigation for the overflow risk described above — set it on any host that relies on snapshots during backups.

[↑ Table of Contents](#table-of-contents)

---

## 7. Complete Snapshot-Backed Backup Workflow

The snapshot is a **tool for consistency**, not a backup by itself. The workflow is:

```
1. Create LVM snapshot (instant)
2. Mount snapshot read-only
3. Run backup tool against mounted snapshot
4. Unmount snapshot
5. Remove snapshot
```

All examples snapshot the lab volume `backupvg/datalv` (`/data`) and write the backup to `/backup`. For a system volume, substitute e.g. `/dev/rhel/root` — the workflow is identical.

### 7.1 Snapshot + xfsdump

```bash
sudo lvcreate -L 1G -s -n data_snap /dev/backupvg/datalv
sudo mkdir -p /mnt/data_snap
sudo mount -o ro,nouuid /dev/backupvg/data_snap /mnt/data_snap

sudo xfsdump -l 0 \
  -L "data-snap-$(date +%Y%m%d)" \
  -M "data-snap-media" \
  -f /backup/data-snap-$(date +%Y%m%d).xfsdump \
  /mnt/data_snap

sudo umount /mnt/data_snap
sudo lvremove -f /dev/backupvg/data_snap
```

### 7.2 Snapshot + rsync

```bash
sudo lvcreate -L 1G -s -n data_snap /dev/backupvg/datalv
sudo mkdir -p /mnt/data_snap
sudo mount -o ro,nouuid /dev/backupvg/data_snap /mnt/data_snap

sudo rsync -aAXvh --delete --numeric-ids \
  /mnt/data_snap/ \
  /backup/data-$(date +%Y-%m-%d)/

sudo umount /mnt/data_snap
sudo lvremove -f /dev/backupvg/data_snap
```

### 7.3 Snapshot + tar

```bash
sudo lvcreate -L 1G -s -n data_snap /dev/backupvg/datalv
sudo mkdir -p /mnt/data_snap
sudo mount -o ro,nouuid /dev/backupvg/data_snap /mnt/data_snap

sudo tar -czpf /backup/configs-snap-$(date +%Y%m%d).tar.gz \
  --xattrs --acls --selinux --numeric-owner \
  -C /mnt/data_snap \
  configs/

sudo umount /mnt/data_snap
sudo lvremove -f /dev/backupvg/data_snap
```

[↑ Table of Contents](#table-of-contents)

---

## 8. Production Snapshot Backup Script

```bash
sudo tee /usr/local/bin/lvm-snapshot-backup.sh <<'SCRIPT'
#!/bin/bash
# lvm-snapshot-backup.sh — LVM snapshot-based backup
# Creates snapshot, backs up via rsync, removes snapshot

set -euo pipefail

LOG_FILE="/var/log/lvm-snapshot-backup.log"
BACKUP_ROOT="/backup/snapshots"
SNAP_MOUNT="/mnt/lvm_snap"
DATE=$(date +%Y-%m-%d)
HOST=$(hostname -s)

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "${LOG_FILE}"; }

# Define source LVs and their config:
# Format: "VG/LV:MOUNT:SNAP_SIZE_GB:BACKUP_NAME"
VOLUMES=(
  "backupvg/datalv:/data:1G:data"
  # System-VG examples — require free space in the 'rhel' VG:
  # "rhel/root:/:5G:root"
  # "rhel/home:/home:3G:home"
)

# Cleanup function - always runs on exit
cleanup() {
  local SNAP_LV="${1:-}"
  if [[ -n "${SNAP_LV}" ]]; then
    log "Cleanup: unmounting and removing snapshot ${SNAP_LV}"
    umount "${SNAP_MOUNT}" 2>/dev/null || true
    lvremove -f "${SNAP_LV}" 2>/dev/null || true
  fi
}

mkdir -p "${BACKUP_ROOT}/${HOST}/${DATE}" "${SNAP_MOUNT}"

log "=== LVM snapshot backup START: ${HOST} ${DATE} ==="

for VOLDEF in "${VOLUMES[@]}"; do
  IFS=: read -r VGLV MOUNT SNAPSIZE BACKUPNAME <<< "${VOLDEF}"

  VG=$(dirname "${VGLV}")
  LV=$(basename "${VGLV}")
  SNAP_NAME="${LV}_snap_$$"
  SNAP_LV="/dev/${VG}/${SNAP_NAME}"
  DEST="${BACKUP_ROOT}/${HOST}/${DATE}/${BACKUPNAME}"
  PREV_DEST="${BACKUP_ROOT}/${HOST}/$(date -d yesterday +%Y-%m-%d)/${BACKUPNAME}"

  # Register cleanup trap for this snapshot
  trap "cleanup ${SNAP_LV}" EXIT

  log "Creating snapshot: ${VGLV} → ${SNAP_LV} (${SNAPSIZE})"
  lvcreate -L "${SNAPSIZE}" -s -n "${SNAP_NAME}" "/dev/${VGLV}" \
    || { log "ERROR: snapshot creation failed for ${VGLV}"; continue; }

  log "Mounting snapshot at ${SNAP_MOUNT}"
  mount -o ro,nouuid "${SNAP_LV}" "${SNAP_MOUNT}" \
    || { log "ERROR: mount failed for ${SNAP_LV}"; lvremove -f "${SNAP_LV}"; continue; }

  log "Checking snapshot health"
  SNAP_PCT=$(lvs --noheadings -o snap_percent "${SNAP_LV}" | tr -d ' ')
  log "Snapshot usage: ${SNAP_PCT}%"

  log "Backing up ${SNAP_MOUNT}/${BACKUPNAME} → ${DEST}"
  mkdir -p "${DEST}"
  rsync -aAXh --delete --numeric-ids \
    --link-dest="${PREV_DEST}" \
    "${SNAP_MOUNT}/" \
    "${DEST}/" \
    2>>"${LOG_FILE}" && log "OK: ${BACKUPNAME}" || log "ERROR: rsync failed for ${BACKUPNAME}"

  log "Unmounting snapshot"
  umount "${SNAP_MOUNT}"

  log "Removing snapshot ${SNAP_LV}"
  lvremove -f "${SNAP_LV}"

  # Clear the trap for this volume
  trap - EXIT
done

# Retention: keep 14 days
log "Pruning snapshots older than 14 days"
find "${BACKUP_ROOT}/${HOST}" -maxdepth 1 -type d -name "????-??-??" \
  | sort | head -n -14 | xargs -r rm -rf

log "=== LVM snapshot backup END ==="
SCRIPT

sudo chmod +x /usr/local/bin/lvm-snapshot-backup.sh
```

[↑ Table of Contents](#table-of-contents)

---

## 9. Database-Consistent Snapshots

For databases, combine a pre-freeze step with the LVM snapshot. These are **production examples** — they snapshot the system VG where the database files (`/var/lib/mysql`, `/var/lib/pgsql`) actually live; substitute your VG/LV names.

### MariaDB/MySQL

> **⚠️ The lock must be held in ONE open session.** `FLUSH TABLES WITH READ LOCK` is released the instant the client that took it disconnects — running it as a one-shot `mysql -e` command locks nothing by the time you snapshot.

```bash
# 1. Open an interactive session and take the lock — LEAVE THIS SESSION OPEN
mysql -u root
#   mysql> FLUSH TABLES WITH READ LOCK;

# 2. In a second terminal, while the first session still holds the lock:
sudo lvcreate -L 5G -s -n mysql_snap /dev/rhel/root

# 3. Back in the first session, release the lock and exit
#   mysql> UNLOCK TABLES;
#   mysql> \q

# 4. Mount and back up
sudo mount -o ro,nouuid /dev/rhel/mysql_snap /mnt/mysql_snap
sudo rsync -aAXh /mnt/mysql_snap/var/lib/mysql/ /backup/mysql-$(date +%Y%m%d)/
sudo umount /mnt/mysql_snap
sudo lvremove -f /dev/rhel/mysql_snap
```

### Better approach: use mariabackup

```bash
# For online backups of MariaDB without locking, use mariabackup
sudo dnf install -y mariadb-backup

sudo mariabackup --backup \
  --target-dir=/backup/mariadb-$(date +%Y%m%d) \
  --user=root

# Then take an LVM snapshot of the prepared backup directory
```

### PostgreSQL

> **Note:** `pg_start_backup()`/`pg_stop_backup()` were **removed in PostgreSQL 15** (RHEL 10 ships PostgreSQL 16+). The replacements `pg_backup_start()`/`pg_backup_stop()` are session-scoped — both calls must run in the **same open session**, so they cannot be split across separate `psql -c` invocations. For most cases, simply use `pg_basebackup` instead.

```bash
# 1. Open ONE psql session and start backup mode — leave the session open
sudo -u postgres psql
#   postgres=# SELECT pg_backup_start('lvm-snapshot');

# 2. In a second terminal, create the LVM snapshot
sudo lvcreate -L 5G -s -n pg_snap /dev/rhel/root

# 3. Back in the same psql session, stop backup mode and exit
#   postgres=# SELECT pg_backup_stop();
#   postgres=# \q

# 4. Mount and back up snapshot
sudo mount -o ro,nouuid /dev/rhel/pg_snap /mnt/pg_snap
sudo rsync -aAXh /mnt/pg_snap/var/lib/pgsql/ /backup/pgsql-$(date +%Y%m%d)/
sudo umount /mnt/pg_snap
sudo lvremove -f /dev/rhel/pg_snap
```

[↑ Table of Contents](#table-of-contents)

---

## 10. Restoring from an LVM Snapshot (Direct Method)

In addition to backing up to a secondary location, you can restore directly from a snapshot:

```bash
# If the snapshot is still mounted (not yet removed):
# Restore a single file from the snapshot
sudo cp /mnt/data_snap/configs/hosts /data/configs/hosts

# Or use rsync to restore a directory
sudo rsync -aAXh /mnt/data_snap/configs/ /data/configs/

# ⚠️ WARNING: Merging a snapshot back into the origin LV replaces ALL data
# Only do this for full roll-back scenarios, not partial restores
sudo umount /mnt/data_snap /data        # origin must not be in use to merge now
sudo lvconvert --merge /dev/backupvg/data_snap
sudo mount /data                        # remount the rolled-back volume
# If the origin is in use (e.g. the root filesystem), the merge is deferred
# to the next activation — i.e. the next reboot. For an origin you can
# unmount, the merge starts immediately (umount, then lvconvert --merge).
```

[↑ Table of Contents](#table-of-contents)

---

## Lab Exercises

### Lab 05-1: Create and inspect an LVM snapshot

```bash
# 1. Find your LVM layout
sudo pvs && sudo vgs && sudo lvs

# 2. Confirm free space in the lab VG (~2G after the README setup)
sudo vgs --units g backupvg

# 3. Create a snapshot of the /data volume
sudo lvcreate -L 1G -s -n test_snap /dev/backupvg/datalv

# 4. View snapshot details
sudo lvs -o lv_name,lv_size,snap_percent,origin,lv_attr

# 5. Check CoW usage (should be low — just created)
sudo lvs --noheadings -o snap_percent /dev/backupvg/test_snap
```

### Lab 05-2: Mount snapshot and verify consistency

```bash
# 1. Mount the snapshot
sudo mkdir -p /mnt/test_snap
sudo mount -o ro,nouuid /dev/backupvg/test_snap /mnt/test_snap

# 2. Create a file in the ORIGINAL /data
echo "this was written after snapshot" | sudo tee /data/after-snapshot-file.txt

# 3. Check the snapshot — the new file should NOT appear
ls /mnt/test_snap/   # after-snapshot-file.txt should be absent

# 4. Check original
ls /data/            # after-snapshot-file.txt should be present
```

### Lab 05-3: Back up from snapshot using rsync

```bash
# Backup from the mounted snapshot
sudo rsync -aAXvh --delete --numeric-ids \
  /mnt/test_snap/ \
  /backup/data-snap-$(date +%Y-%m-%d)/

# Verify
ls /backup/data-snap-$(date +%Y-%m-%d)/
```

### Lab 05-4: Monitor and clean up

```bash
# 1. Check snapshot fill percentage
sudo lvs -o lv_name,snap_percent /dev/backupvg/test_snap

# 2. Write some data to /data to see CoW space increase
sudo dd if=/dev/urandom bs=1M count=100 of=/data/cow_test.bin

# 3. Check snapshot fill again
sudo lvs -o lv_name,snap_percent /dev/backupvg/test_snap

# 4. Unmount and remove
sudo umount /mnt/test_snap
sudo lvremove -f /dev/backupvg/test_snap
sudo lvs  # Confirm snapshot is gone
sudo rm -f /data/cow_test.bin /data/after-snapshot-file.txt   # tidy the lab volume
```

### Lab 05-5: Run the production snapshot backup script

```bash
# The script targets backupvg/datalv by default — edit VOLUMES if your
# layout differs (e.g. enable the rhel/root example lines)
sudo nano /usr/local/bin/lvm-snapshot-backup.sh

# Run it
sudo /usr/local/bin/lvm-snapshot-backup.sh

# Review log
sudo cat /var/log/lvm-snapshot-backup.log
```

[↑ Table of Contents](#table-of-contents)

---

## Review Questions

1. What is the Copy-on-Write mechanism in LVM snapshots?
2. Why do LVM snapshots not require copying all data when created?
3. What happens if a thick snapshot's CoW space fills up?
4. Why must you use the `nouuid` mount option for XFS snapshots?
5. What is the difference between a thick and a thin LVM snapshot?
6. What is the correct workflow for using an LVM snapshot as a pre-backup consistency mechanism?
7. How do you check how full a snapshot's CoW space is?
8. What does `lvconvert --merge` do, and when should it be used?
9. How do you ensure a MariaDB database is consistent before taking an LVM snapshot?
10. What command creates a 1GB snapshot of `/dev/backupvg/datalv` named `data_backup`?

[↑ Table of Contents](#table-of-contents)

---

## Answers to Review Questions

1. **Copy-on-Write (CoW):** When the original LV is about to overwrite a block, LVM first copies the original block into the snapshot's reserved space, then allows the write to proceed. The snapshot always contains the pre-change versions of blocks, reflecting the state at the moment of creation.
2. The snapshot starts empty. It only stores blocks that have been modified in the **original** LV since the snapshot was created. Unmodified blocks are read directly from the original LV, sharing the physical storage. This is why a 100GB LV can have a 5GB snapshot — if less than 5GB of data changes.
3. If the CoW space fills 100%, the snapshot becomes **invalid** — it can no longer store pre-change block copies and becomes inconsistent. It must be removed; any backup in progress from it is unreliable. Prevent this by allocating sufficient space and monitoring `snap_percent`.
4. XFS stores its UUID inside the filesystem metadata. When you snapshot an XFS volume, the snapshot has the **same UUID** as the original. Linux's kernel refuses to simultaneously mount two XFS volumes with the same UUID. `nouuid` overrides this check for the snapshot mount.
5. **Thick snapshot:** allocates a fixed CoW space up front from VG free extents. **Thin snapshot:** requires a thin pool; CoW space is allocated dynamically from the pool as needed. Thin snapshots are more flexible and share pool space efficiently when multiple snapshots exist simultaneously.
6. 1) Create snapshot → 2) Mount snapshot read-only → 3) Run backup tool against snapshot mount → 4) Unmount snapshot → 5) Remove snapshot LV.
7. `lvs -o lv_name,snap_percent` — shows the percentage of CoW space used. Also `lvs --noheadings -o snap_percent /dev/VG/SNAP`.
8. `lvconvert --merge SNAP_LV` merges the snapshot back into the origin LV, replacing its content. If the origin is in use (e.g. the root filesystem) the merge is deferred to the next activation/reboot; an unmounted origin merges immediately. Used for rolling back a volume to a previous state. Not for partial restores — it replaces all data.
9. In an **interactive MariaDB session that stays open**, issue `FLUSH TABLES WITH READ LOCK;` (the lock dies with the session, so a one-shot `mysql -e` will not work). While that session holds the lock, create the LVM snapshot from a second terminal, then release with `UNLOCK TABLES;` in the same session. The lock window is only as long as the `lvcreate` takes (typically under a second).
10. `sudo lvcreate -L 1G -s -n data_backup /dev/backupvg/datalv`

[↑ Table of Contents](#table-of-contents)

---

*Previous: [04 — dump / xfsdump](04-dump-restore.md)*
*Next: [06 — Restic](06-restic.md)*

© 2026 UncleJS — Licensed under CC BY-NC-SA 4.0
