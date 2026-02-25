# Module 05 — LVM Snapshots — Consistent Pre-Backup Freezes

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

# Common RHEL 10 LVM layout
# VG: rhel
# LVs: rhel-root (/) and rhel-swap
```

### Available free space in the Volume Group

The snapshot needs free space in the same VG as the source LV.

```bash
# Check free space in each VG
vgs --units g

# Example output:
#   VG    #PV #LV #SN Attr   VSize  VFree
#   rhel    1   2   0 wz--n- 99.00g 10.00g
```

You need at least **10–20% of the source LV size** as free space for the snapshot. More is better if the system is write-heavy.

[↑ Table of Contents](#table-of-contents)

---

## 4. Thick Snapshots

Thick snapshots are the traditional LVM snapshot type. They reserve space up front from the VG's free extents.

### 4.1 Creating a thick snapshot

```bash
# Syntax: lvcreate -L SIZE -s -n SNAPNAME SOURCE_LV

# Snapshot the root LV (adjust VG/LV names to match yours)
sudo lvcreate -L 5G -s -n root_snap /dev/rhel/root

# Snapshot /home
sudo lvcreate -L 3G -s -n home_snap /dev/rhel/home

# Verify snapshot was created
sudo lvs
# Look for Attr column showing "swi-a-s--" (s = snapshot, w = writable, i = inherited)
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

# Mount the snapshot read-only
sudo mount -o ro /dev/rhel/root_snap /mnt/snapshot

# For XFS: add 'nouuid' to handle duplicate UUID
sudo mount -o ro,nouuid /dev/rhel/root_snap /mnt/snapshot

# Verify
df -h /mnt/snapshot
ls /mnt/snapshot/etc/
```

**Why `nouuid`?** XFS embeds a UUID in the filesystem. The snapshot has the same UUID as the original. Linux will refuse to mount two XFS filesystems with the same UUID without the `nouuid` option.

### 4.4 Removing a snapshot

```bash
# Unmount first
sudo umount /mnt/snapshot

# Remove the snapshot LV
sudo lvremove -f /dev/rhel/root_snap

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
# Create a thin pool LV in the VG
# (Requires free space in the VG)
sudo lvcreate -L 20G --thinpool thinpool rhel

# Verify
sudo lvs
# thinpool shows as "twi---tz--"
```

### 5.3 Creating a thin LV and thin snapshot

```bash
# Create a thin LV in the pool
sudo lvcreate -V 10G --thin -n thindata rhel/thinpool

# Format and mount the thin LV
sudo mkfs.xfs /dev/rhel/thindata
sudo mkdir -p /data/thin
sudo mount /dev/rhel/thindata /data/thin

# Create a thin snapshot (no size required — uses pool space dynamically)
sudo lvcreate -s --name thindata_snap rhel/thindata

# Mount the thin snapshot
sudo mount -o ro,nouuid /dev/rhel/thindata_snap /mnt/snapshot
```

### 5.4 Monitoring thin pool usage

```bash
# Check thin pool data and metadata usage
sudo lvs -a rhel/thinpool

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
sudo lvs -o lv_name,snap_percent --noheadings | grep -v ""

# Alert if snapshot is > 70% full
sudo lvs -o lv_name,snap_percent --noheadings | awk '$2 > 70 {print "WARNING: "$1" is "$2"% full"}'
```

### How much space to allocate?

A general rule: allocate **15–20% of the source LV size** for the snapshot, or estimate based on your write rate:

```bash
# How many writes happen per minute on the LV?
# Check iostat for the device
iostat -x 1 10 | grep $(lvs --noheadings -o devices /dev/rhel/root | tr -d ' ')

# Conservative approach: use thin snapshots so overflow is automatically managed
```

### Extending a thick snapshot that is filling up

```bash
# Extend the snapshot CoW space (add 2GB more)
sudo lvextend -L +2G /dev/rhel/root_snap

# Or resize to a specific size
sudo lvextend -L 8G /dev/rhel/root_snap
```

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

### 7.1 Snapshot + xfsdump

```bash
sudo lvcreate -L 5G -s -n root_snap /dev/rhel/root
sudo mkdir -p /mnt/root_snap
sudo mount -o ro,nouuid /dev/rhel/root_snap /mnt/root_snap

sudo xfsdump -l 0 \
  -L "root-snap-$(date +%Y%m%d)" \
  -M "root-snap-media" \
  -f /backup/root-snap-$(date +%Y%m%d).xfsdump \
  /mnt/root_snap

sudo umount /mnt/root_snap
sudo lvremove -f /dev/rhel/root_snap
```

### 7.2 Snapshot + rsync

```bash
sudo lvcreate -L 5G -s -n home_snap /dev/rhel/home
sudo mkdir -p /mnt/home_snap
sudo mount -o ro,nouuid /dev/rhel/home_snap /mnt/home_snap

sudo rsync -aAXvh --delete --numeric-ids \
  /mnt/home_snap/ \
  /backup/home-$(date +%Y-%m-%d)/

sudo umount /mnt/home_snap
sudo lvremove -f /dev/rhel/home_snap
```

### 7.3 Snapshot + tar

```bash
sudo lvcreate -L 5G -s -n etc_snap /dev/rhel/root
sudo mkdir -p /mnt/etc_snap
sudo mount -o ro,nouuid /dev/rhel/etc_snap /mnt/etc_snap

sudo tar -czpf /backup/etc-snap-$(date +%Y%m%d).tar.gz \
  --xattrs --acls --selinux --numeric-owner \
  -C /mnt/etc_snap \
  etc/

sudo umount /mnt/etc_snap
sudo lvremove -f /dev/rhel/etc_snap
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
  "rhel/root:/:5G:root"
  "rhel/home:/home:3G:home"
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

For databases, combine a pre-freeze step with the LVM snapshot:

### MariaDB/MySQL

```bash
# 1. Flush and lock tables (holds write lock briefly)
mysql -u root -e "FLUSH TABLES WITH READ LOCK;"

# 2. In a second terminal: create snapshot (must be fast)
sudo lvcreate -L 5G -s -n mysql_snap /dev/rhel/root

# 3. Release the lock
mysql -u root -e "UNLOCK TABLES;"

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

```bash
# 1. Start backup mode (writes backup label file)
psql -U postgres -c "SELECT pg_start_backup('lvm-snapshot-$(date +%Y%m%d)');"

# 2. Create LVM snapshot
sudo lvcreate -L 5G -s -n pg_snap /dev/rhel/root

# 3. Stop backup mode
psql -U postgres -c "SELECT pg_stop_backup();"

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
sudo cp /mnt/root_snap/etc/nginx/nginx.conf /etc/nginx/nginx.conf

# Or use rsync to restore a directory
sudo rsync -aAXh /mnt/root_snap/etc/nginx/ /etc/nginx/

# ⚠️ WARNING: Merging a snapshot back into the origin LV replaces ALL data
# Only do this for full roll-back scenarios, not partial restores
sudo lvconvert --merge /dev/rhel/root_snap
# This requires rebooting — the merge happens on next boot
```

[↑ Table of Contents](#table-of-contents)

---

## Lab Exercises

### Lab 05-1: Create and inspect an LVM snapshot

```bash
# 1. Find your LVM layout
sudo pvs && sudo vgs && sudo lvs

# 2. Confirm free space in the VG
sudo vgs --units g

# 3. Create a snapshot of /home (or root if no /home LV)
# Replace 'rhel' and 'home' with your VG and LV names
sudo lvcreate -L 2G -s -n test_snap /dev/rhel/home

# 4. View snapshot details
sudo lvs -o lv_name,lv_size,snap_percent,origin,lv_attr

# 5. Check CoW usage (should be low — just created)
sudo lvs --noheadings -o snap_percent /dev/rhel/test_snap
```

### Lab 05-2: Mount snapshot and verify consistency

```bash
# 1. Mount the snapshot
sudo mkdir -p /mnt/test_snap
sudo mount -o ro,nouuid /dev/rhel/test_snap /mnt/test_snap

# 2. Create a file in the ORIGINAL /home
sudo touch /home/after-snapshot-file.txt
echo "this was written after snapshot" | sudo tee /home/after-snapshot-file.txt

# 3. Check the snapshot — the new file should NOT appear
ls /mnt/test_snap/   # after-snapshot-file.txt should be absent

# 4. Check original
ls /home/            # after-snapshot-file.txt should be present
```

### Lab 05-3: Back up from snapshot using rsync

```bash
# Backup from the mounted snapshot
sudo rsync -aAXvh --delete --numeric-ids \
  /mnt/test_snap/ \
  /backup/home-snap-$(date +%Y-%m-%d)/

# Verify
ls /backup/home-snap-$(date +%Y-%m-%d)/
```

### Lab 05-4: Monitor and clean up

```bash
# 1. Check snapshot fill percentage
sudo lvs -o lv_name,snap_percent /dev/rhel/test_snap

# 2. Write some data to /home to see CoW space increase
dd if=/dev/urandom bs=1M count=100 of=/home/cow_test.bin

# 3. Check snapshot fill again
sudo lvs -o lv_name,snap_percent /dev/rhel/test_snap

# 4. Unmount and remove
sudo umount /mnt/test_snap
sudo lvremove -f /dev/rhel/test_snap
sudo lvs  # Confirm snapshot is gone
```

### Lab 05-5: Run the production snapshot backup script

```bash
# Edit the script to match your VG/LV names
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
10. What command creates a 5GB snapshot of `/dev/rhel/home` named `home_backup`?

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
8. `lvconvert --merge SNAP_LV` marks the snapshot for merging back into the origin LV. On the **next reboot**, the origin LV is replaced entirely with the snapshot's content. Used for rolling back a volume to a previous state. Not for partial restores — it replaces all data.
9. Issue `FLUSH TABLES WITH READ LOCK;` in MariaDB to stop writes and flush dirty pages to disk. Immediately create the LVM snapshot, then release the lock with `UNLOCK TABLES;`. The lock window is only as long as the `lvcreate` command takes (typically under a second).
10. `sudo lvcreate -L 5G -s -n home_backup /dev/rhel/home`

[↑ Table of Contents](#table-of-contents)

---

*Previous: [04 — dump / xfsdump](04-dump-restore.md)*
*Next: [06 — Restic](06-restic.md)*

© 2026 Jaco Steyn — Licensed under CC BY-SA 4.0
