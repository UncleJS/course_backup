# Module 10 — Restore Testing and Disaster Recovery Drills

## Learning Objectives

By the end of this module you will be able to:

- Execute verified restore procedures for every tool covered in this course
- Perform bare-metal recovery from scratch using a backup set
- Build and run a restore test checklist that proves backups are valid
- Design and run regular DR drills with measurable outcomes
- Benchmark restore performance and calculate actual RTO
- Document DR procedures so any team member can execute them
- Identify and fix the most common restore failures before they happen in a crisis

---

## Table of Contents

- [1. Why Test Restores?](#1-why-test-restores)
- [2. Restore Procedure Checklists by Tool](#2-restore-procedure-checklists-by-tool)
  - [2.1 tar restore checklist](#21-tar-restore-checklist)
  - [2.2 rsync restore checklist](#22-rsync-restore-checklist)
  - [2.3 xfsdump / xfsrestore checklist](#23-xfsdump--xfsrestore-checklist)
  - [2.4 LVM snapshot restore checklist](#24-lvm-snapshot-restore-checklist)
  - [2.5 Restic restore checklist](#25-restic-restore-checklist)
  - [2.6 Bareos restore checklist](#26-bareos-restore-checklist)
- [3. Bare-Metal Recovery Procedure](#3-bare-metal-recovery-procedure)
  - [3.1 Prerequisites](#31-prerequisites)
  - [3.2 Bare-metal recovery sequence](#32-bare-metal-recovery-sequence)
  - [3.3 Disk UUID update procedure](#33-disk-uuid-update-procedure)
- [4. Automated Restore Verification](#4-automated-restore-verification)
  - [/usr/local/sbin/verify-backup.sh](#usrlocalsbinverify-backupsh)
- [5. DR Drill Planning](#5-dr-drill-planning)
  - [5.1 Drill types](#51-drill-types)
  - [5.2 DR drill execution checklist](#52-dr-drill-execution-checklist)
  - [5.3 Smoke test suite](#53-smoke-test-suite)
- [6. Restore Performance Benchmarking](#6-restore-performance-benchmarking)
  - [6.1 Benchmark template](#61-benchmark-template)
  - [6.2 Throughput reference targets](#62-throughput-reference-targets)
- [7. Common Restore Failures and Fixes](#7-common-restore-failures-and-fixes)
- [Lab 10 — Restore Drills](#lab-10--restore-drills)
  - [Prerequisites](#prerequisites)
  - [Lab Part A — tar Restore Drill](#lab-part-a--tar-restore-drill)
  - [Lab Part B — Restic Restore Drill](#lab-part-b--restic-restore-drill)
  - [Lab Part C — Automated Verification Script](#lab-part-c--automated-verification-script)
  - [Lab Part D — Restore Benchmark](#lab-part-d--restore-benchmark)
  - [Lab Part E — Smoke Test](#lab-part-e--smoke-test)
- [Review Questions](#review-questions)
- [Answers to Review Questions](#answers-to-review-questions)

---

## 1. Why Test Restores?

> *"A backup you have never restored from is not a backup — it is hope."*

The most common backup failure modes are:

| Failure | How it manifests | Prevention |
|---------|-----------------|------------|
| Silent corruption | Restore fails mid-operation | Weekly `restic check --read-data` |
| Missing files | Critical path not in backup scope | Quarterly full restore test |
| Wrong permissions | Files restored; services won't start | SELinux context + mode testing |
| Stale credentials | Encryption key lost or rotated | Key/password stored in vault |
| Partial snapshots | Backup interrupted; last run incomplete | Check exit codes in logs |
| Schema drift | DB dump restores but app fails | Application smoke test after restore |
| Missing dependencies | Restore succeeds; system won't boot | Boot into restored VM and verify |

**Rule:** Every backup system must have a documented and tested restore procedure. Test it at least:

- After initial setup
- After any configuration change
- Monthly (automated) — file-level integrity check
- Quarterly — full bare-metal or VM-level restore drill

[↑ Table of Contents](#table-of-contents)

---

## 2. Restore Procedure Checklists by Tool

### 2.1 tar restore checklist

```
[ ] Identify the correct archive file (date, label)
[ ] Verify archive integrity:     tar -tzf archive.tar.gz > /dev/null
[ ] Check available disk space:   df -h /restore/target
[ ] Restore to staging area first (never directly to production)
[ ] Confirm SELinux labels preserved: ls -Z /restore/target/etc/passwd
[ ] Check file ownership: ls -ln /restore/target/etc/shadow
[ ] Validate checksum if stored: sha256sum -c archive.tar.gz.sha256
[ ] Test critical file presence: /etc/passwd, /etc/fstab, application configs
[ ] If replacing live data: stop services, swap, restart, smoke test
```

**Restore command reference:**

```bash
# List archive contents
tar -tzf /mnt/backup/etc-20260224.tar.gz | head -50

# Test extraction (dry-run)
tar -tzf /mnt/backup/etc-20260224.tar.gz > /dev/null && echo "Archive OK"

# Full extract preserving xattrs and SELinux
tar -xzf /mnt/backup/etc-20260224.tar.gz \
    --xattrs --xattrs-include='*' \
    --selinux \
    --acls \
    -C /restore/staging/

# Extract single file
tar -xzf /mnt/backup/etc-20260224.tar.gz \
    -C /restore/staging/ \
    etc/nginx/nginx.conf

# Extract incremental chain (must apply in order)
tar -xzf /mnt/backup/etc-full-20260217.tar.gz   -C /restore/staging/ \
    --xattrs --xattrs-include='*' --selinux --acls
tar -xzf /mnt/backup/etc-inc-20260218.tar.gz    -C /restore/staging/ \
    --xattrs --xattrs-include='*' --selinux --acls
tar -xzf /mnt/backup/etc-inc-20260219.tar.gz    -C /restore/staging/ \
    --xattrs --xattrs-include='*' --selinux --acls
# ... continue through the chain to the desired date
```

[↑ Table of Contents](#table-of-contents)

---

### 2.2 rsync restore checklist

```
[ ] Identify snapshot directory (date-stamped path under latest/)
[ ] Verify snapshot directory exists and is non-empty
[ ] Check hardlink integrity: du -sh snapshot/ vs du --apparent-size -sh snapshot/
[ ] Restore to staging first
[ ] Confirm permissions and SELinux labels intact
[ ] Stop target services before live cutover
[ ] rsync from snapshot back to live path (reverse direction)
[ ] Restart services and run smoke tests
```

**Restore command reference:**

```bash
# List available snapshots
ls -lt /mnt/backup/rsync/backup-server/ | head -20

# Restore specific snapshot to staging
rsync -aAXH --progress \
    /mnt/backup/rsync/backup-server/2026/02/24_020000/ \
    /restore/staging/

# Restore single directory from snapshot
rsync -aAXH \
    /mnt/backup/rsync/backup-server/latest/etc/nginx/ \
    /restore/staging/nginx/

# Live cutover (stop service first!)
systemctl stop nginx
rsync -aAXH --delete \
    /mnt/backup/rsync/backup-server/latest/etc/nginx/ \
    /etc/nginx/
systemctl start nginx
systemctl status nginx
```

[↑ Table of Contents](#table-of-contents)

---

### 2.3 xfsdump / xfsrestore checklist

```
[ ] Identify dump files (level 0 full + level 1..N incrementals)
[ ] Verify dump inventory: xfsrestore -I
[ ] Confirm target filesystem is XFS and mounted
[ ] Always restore level 0 first, then incrementals in order
[ ] After restore: verify with xfs_repair -n (read-only check)
[ ] Relabel SELinux contexts: restorecon -Rv /restore/target
```

**Restore command reference:**

```bash
# Show dump inventory
xfsrestore -I

# Restore full (level 0) dump
xfsrestore -f /mnt/backup/xfsdump/data.level0 /restore/target/

# Restore incremental (level 1) on top of level 0
xfsrestore -f /mnt/backup/xfsdump/data.level1 /restore/target/

# Restore single file interactively
xfsrestore -i -f /mnt/backup/xfsdump/data.level0 /restore/target/
# At prompt: ls, cd, add <file>, extract

# Non-interactive single file
xfsrestore -f /mnt/backup/xfsdump/data.level0 \
    -s etc/fstab \
    /restore/target/

# Verify XFS filesystem integrity (read-only)
xfs_repair -n /dev/backupvg/data
```

[↑ Table of Contents](#table-of-contents)

---

### 2.4 LVM snapshot restore checklist

```
[ ] Confirm snapshot volume still exists (not expired/overflowed)
[ ] Check snapshot space usage: lvs --options lv_name,snap_percent
[ ] Mount snapshot read-only to inspect contents
[ ] Copy required files from snapshot to live filesystem
[ ] OR: merge snapshot back into origin (destructive — read carefully)
[ ] Remove snapshot after restore is confirmed
```

**Restore command reference:**

```bash
# Check snapshot status (snap_percent < 100 = valid)
lvs -o lv_name,lv_attr,snap_percent backupvg

# Mount snapshot read-only
mkdir -p /mnt/snap_restore
mount -o ro,nouuid /dev/backupvg/data_snap /mnt/snap_restore

# Copy specific files out
cp -a /mnt/snap_restore/var/lib/mysql/mydb/ /var/lib/mysql/mydb_restored/

# Unmount when done
umount /mnt/snap_restore

# Merge snapshot into origin (DESTRUCTIVE — reverts origin to snapshot state)
# WARNING: origin LV must be inactive (unmounted) for merge
umount /mnt/data
lvconvert --merge /dev/backupvg/data_snap
# After reboot or re-activation, origin LV reflects snapshot state
lvchange -an /dev/backupvg/data
lvchange -ay /dev/backupvg/data
mount /dev/backupvg/data /mnt/data

# Remove snapshot without merging
lvremove -f /dev/backupvg/data_snap
```

[↑ Table of Contents](#table-of-contents)

---

### 2.5 Restic restore checklist

```
[ ] List snapshots and identify target snapshot ID
[ ] Run integrity check before restore: restic check
[ ] Restore to staging directory first
[ ] Verify file tree, permissions, SELinux labels
[ ] Confirm data completeness (spot-check critical files)
[ ] For live cutover: stop services, restore, relabel, restart
[ ] Run application smoke tests
[ ] Tag restored snapshot: restic tag --add "restored-$(date +%Y%m%d)"
```

**Restore command reference:**

```bash
# List all snapshots
restic --repo /mnt/backup/restic \
    --password-file /etc/backup/restic-password \
    snapshots

# List files inside a snapshot
restic --repo /mnt/backup/restic \
    --password-file /etc/backup/restic-password \
    ls latest

# Restore latest snapshot to staging
restic --repo /mnt/backup/restic \
    --password-file /etc/backup/restic-password \
    restore latest \
    --target /restore/staging/

# Restore specific snapshot
restic --repo /mnt/backup/restic \
    --password-file /etc/backup/restic-password \
    restore abc12345 \
    --target /restore/staging/

# Restore only a subset of paths
restic --repo /mnt/backup/restic \
    --password-file /etc/backup/restic-password \
    restore latest \
    --include /etc/nginx \
    --target /restore/staging/

# Mount snapshot as a FUSE filesystem for interactive browsing
restic --repo /mnt/backup/restic \
    --password-file /etc/backup/restic-password \
    mount /mnt/restic_mount &
ls /mnt/restic_mount/snapshots/latest/
# Unmount when done
fusermount -u /mnt/restic_mount
```

[↑ Table of Contents](#table-of-contents)

---

### 2.6 Bareos restore checklist

```
[ ] Open bconsole and confirm storage daemon and director are running
[ ] Use "restore" command and select job/client/date
[ ] Review file selection before submitting
[ ] Monitor restore job: status client=backup-client
[ ] Check restore log: list joblog jobid=<id>
[ ] Verify restored files on client
[ ] Relabel SELinux contexts if needed: restorecon -Rv /restore/target
```

**Restore command reference (bconsole):**

```
*status director
*status storage
*list jobs
*restore
  (follow wizard: select client, date range, file path)
*status client=backup-client-fd
*list joblog jobid=42
*quit
```

[↑ Table of Contents](#table-of-contents)

---

## 3. Bare-Metal Recovery Procedure

Bare-metal recovery is the process of restoring a machine from nothing — no OS, potentially different hardware. This is the ultimate test of a backup system.

### 3.1 Prerequisites

- Bootable recovery media (RHEL 10 install ISO, rescue USB)
- Network access to the backup server
- Root password or SSH key for backup server
- Written copy of partition layout and mount points
- Restic password / encryption keys (printed or stored offline)

### 3.2 Bare-metal recovery sequence

```
Phase 1: Boot and assess
  1. Boot from RHEL 10 rescue media
  2. Select "Rescue a RHEL system" → "Skip to shell"
  3. Assess disk layout: lsblk, fdisk -l

Phase 2: Recreate partition/LVM layout
  4. Partition disk to match original:
     fdisk /dev/sda   (or use gdisk for GPT)
  5. Recreate VGs and LVs:
     pvcreate /dev/sda2
     vgcreate rhel /dev/sda2
     lvcreate -n root -L 40G rhel
     lvcreate -n swap -L 4G  rhel
     lvcreate -n home -L 20G rhel
  6. Format filesystems:
     mkfs.xfs /dev/rhel/root
     mkfs.xfs /dev/rhel/home
     mkswap   /dev/rhel/swap
  7. Mount target:
     mount /dev/rhel/root /mnt/sysroot
     mkdir -p /mnt/sysroot/{boot,home,etc}
     mount /dev/sda1      /mnt/sysroot/boot
     mount /dev/rhel/home /mnt/sysroot/home

Phase 3: Restore from backup
  8. Install restic on rescue system (or use statically-linked binary):
     curl -Lo /usr/local/bin/restic \
         https://github.com/restic/restic/releases/latest/download/restic_linux_amd64
     chmod +x /usr/local/bin/restic
  9. Restore from restic repository:
     export RESTIC_PASSWORD="your-password"
     restic --repo sftp:backupuser@192.168.100.10:/backup/restic \
         restore latest \
         --target /mnt/sysroot/
 10. Restore /boot separately (may be separate archive):
     tar -xzf /mnt/backup/boot-latest.tar.gz -C /mnt/sysroot/boot/

Phase 4: Rebuild bootloader
 11. Chroot into restored system:
     for d in dev proc sys run; do
         mount --bind /$d /mnt/sysroot/$d
     done
     chroot /mnt/sysroot /bin/bash
 12. Reinstall GRUB:
     grub2-install /dev/sda
     grub2-mkconfig -o /boot/grub2/grub.cfg
 13. Rebuild initramfs:
     dracut -f --regenerate-all
 14. Verify /etc/fstab UUIDs match current disk UUIDs:
     blkid
     # Update /etc/fstab if UUIDs changed (new hardware or reformatted disks)

Phase 5: Relabel and boot
 15. Create SELinux autorelabel trigger:
     touch /mnt/sysroot/.autorelabel
 16. Exit chroot and unmount:
     exit
     umount -R /mnt/sysroot
 17. Reboot — system will relabel on first boot (takes 5–15 min)
```

### 3.3 Disk UUID update procedure

When restoring to different hardware or reformatted disks:

```bash
# On chrooted system — get new UUIDs
blkid

# Update /etc/fstab (replace OLD-UUID with new values from blkid output)
# Example: replace UUID= lines
sed -i 's/UUID=old-root-uuid/UUID=new-root-uuid/' /etc/fstab
sed -i 's/UUID=old-boot-uuid/UUID=new-boot-uuid/' /etc/fstab

# Update GRUB config
grub2-mkconfig -o /boot/grub2/grub.cfg
```

[↑ Table of Contents](#table-of-contents)

---

## 4. Automated Restore Verification

Manual restore testing is necessary but not sufficient. Automate file-level verification to run after every backup.

### /usr/local/sbin/verify-backup.sh

```bash
#!/usr/bin/env bash
# Automated backup verification script
# Run after every backup job to confirm the last snapshot is readable.
set -euo pipefail

RESTIC_REPO="${RESTIC_REPO:-/mnt/backup/restic}"
RESTIC_PASSWORD_FILE="${RESTIC_PASSWORD_FILE:-/etc/backup/restic-password}"
VERIFY_TARGET="/tmp/backup-verify-$$"
ALERT_EMAIL="${ALERT_EMAIL:-root@localhost}"
LOG_FILE="/var/log/backup/verify-$(date +%Y%m%d_%H%M%S).log"

exec > >(tee -a "$LOG_FILE") 2>&1

alert() {
    echo "[VERIFY FAIL] $*" | mailx -s "[BACKUP VERIFY FAIL] $(hostname)" "$ALERT_EMAIL" 2>/dev/null || true
    logger -t backup-verify -p local0.err "FAIL: $*"
    exit 1
}

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

# Step 1: Repository integrity check
log "=== Step 1: Repository integrity check ==="
restic --repo "$RESTIC_REPO" --password-file "$RESTIC_PASSWORD_FILE" \
    check --read-data-subset=10% \
    || alert "restic check failed"

# Step 2: Verify latest snapshot is accessible
log "=== Step 2: Snapshot accessibility ==="
SNAP_COUNT=$(restic --repo "$RESTIC_REPO" --password-file "$RESTIC_PASSWORD_FILE" \
    snapshots --json | python3 -c "import sys,json; print(len(json.load(sys.stdin)))")
[[ "$SNAP_COUNT" -gt 0 ]] || alert "No snapshots found in repository"
log "Found $SNAP_COUNT snapshots."

# Step 3: Restore critical files to temp directory
log "=== Step 3: Critical file restore test ==="
mkdir -p "$VERIFY_TARGET"
trap 'rm -rf "$VERIFY_TARGET"' EXIT

restic --repo "$RESTIC_REPO" --password-file "$RESTIC_PASSWORD_FILE" \
    restore latest \
    --include /etc/passwd \
    --include /etc/fstab \
    --include /etc/hostname \
    --target "$VERIFY_TARGET" \
    || alert "Restore of critical files failed"

# Step 4: Verify critical files are present and non-empty
for f in etc/passwd etc/fstab etc/hostname; do
    if [[ ! -s "${VERIFY_TARGET}/${f}" ]]; then
        alert "Critical file missing or empty after restore: $f"
    fi
    log "OK: ${f}"
done

# Step 5: Verify /etc/passwd is a valid passwd file
grep -qE '^root:' "${VERIFY_TARGET}/etc/passwd" \
    || alert "/etc/passwd does not contain root entry after restore"
log "OK: /etc/passwd contains root entry"

log "=== Verification PASSED ==="
logger -t backup-verify -p local0.info "Verification PASSED for $(hostname)"
```

```bash
chmod 700 /usr/local/sbin/verify-backup.sh

# Add to systemd as a post-backup unit
cat > /etc/systemd/system/backup-verify.service << 'EOF'
[Unit]
Description=Verify backup integrity after backup job
After=backup-incremental.service backup-full.service

[Service]
Type=oneshot
User=root
ExecStart=/usr/local/sbin/verify-backup.sh
StandardOutput=journal
StandardError=journal
SyslogIdentifier=backup-verify
EOF

# Trigger verification after each backup service
# Add to backup-incremental.service and backup-full.service:
# ExecStartPost=/usr/local/sbin/verify-backup.sh
```

[↑ Table of Contents](#table-of-contents)

---

## 5. DR Drill Planning

### 5.1 Drill types

| Type | Frequency | Description | Downtime |
|------|-----------|-------------|---------|
| File restore test | Monthly | Restore 5–10 random files; verify contents | None |
| Service restore test | Quarterly | Restore a non-prod service from backup | Minimal |
| VM/container full restore | Quarterly | Full OS restore to a test VM | None (test env) |
| Bare-metal drill | Annually | Full restore to physical hardware | Planned window |
| Tabletop exercise | Annually | Walk through DR runbook without actually restoring | None |

### 5.2 DR drill execution checklist

```
PRE-DRILL
[ ] Schedule maintenance window (if required)
[ ] Notify all stakeholders
[ ] Identify backup set to restore from (do not use production backups destructively)
[ ] Prepare target system (VM, spare hardware, or isolated network)
[ ] Have this runbook printed and available offline
[ ] Have encryption keys and passwords available offline

DRILL EXECUTION
[ ] Record start time: T=0
[ ] Follow bare-metal restore procedure (Section 3) or tool-specific procedure (Section 2)
[ ] Document every deviation from expected procedure
[ ] Record time when OS boots: T=boot
[ ] Record time when services are healthy: T=service_up
[ ] Record time when application smoke tests pass: T=app_ok
[ ] Calculate RTO = T=app_ok - T=0

POST-DRILL
[ ] Compare measured RTO to target RTO
[ ] Document all deviations and failures
[ ] File issues for any gaps discovered
[ ] Update runbook with improvements
[ ] Report results to management/stakeholders
[ ] Schedule next drill
```

### 5.3 Smoke test suite

After bare-metal or full restore, run these checks before declaring recovery complete:

```bash
#!/usr/bin/env bash
# smoke-test.sh — Post-restore validation
set -uo pipefail

PASS=0; FAIL=0
ok()   { echo "[PASS] $*"; ((PASS++)); }
fail() { echo "[FAIL] $*"; ((FAIL++)); }

# OS fundamentals
[[ "$(hostname)" != "" ]]              && ok "hostname set"          || fail "hostname empty"
systemctl is-system-running &>/dev/null && ok "systemd running"      || fail "systemd not fully running"
ping -c1 -W2 8.8.8.8 &>/dev/null       && ok "internet reachable"   || fail "no internet"
ping -c1 -W2 192.168.100.10 &>/dev/null && ok "backup server reachable" || fail "backup server unreachable"

# Filesystem
mountpoint -q /       && ok "/ mounted"    || fail "/ not mounted"
mountpoint -q /home   && ok "/home mounted" || fail "/home not mounted"
[[ "$(df -T / | awk 'NR==2{print $2}')" == "xfs" ]] && ok "root is XFS" || fail "root is not XFS"

# SELinux
sestatus | grep -q "enforcing" && ok "SELinux enforcing" || fail "SELinux not enforcing"

# Critical services (adjust to your environment)
for svc in sshd chronyd rsyslog; do
    systemctl is-active --quiet "$svc" && ok "$svc running" || fail "$svc not running"
done

# User accounts
getent passwd root &>/dev/null && ok "root account exists"  || fail "root account missing"
getent passwd admin &>/dev/null && ok "admin account exists" || fail "admin account missing" 

# Summary
echo ""
echo "Results: ${PASS} passed, ${FAIL} failed"
[[ "$FAIL" -eq 0 ]] && exit 0 || exit 1
```

[↑ Table of Contents](#table-of-contents)

---

## 6. Restore Performance Benchmarking

Measuring restore speed lets you validate your RTO target is achievable.

### 6.1 Benchmark template

```bash
#!/usr/bin/env bash
# benchmark-restore.sh
set -uo pipefail

REPO="${RESTIC_REPO:-/mnt/backup/restic}"
PASSFILE="${RESTIC_PASSWORD_FILE:-/etc/backup/restic-password}"
TARGET="/tmp/restore-bench-$$"
mkdir -p "$TARGET"
trap 'rm -rf "$TARGET"' EXIT

log() { echo "[$(date '+%H:%M:%S')] $*"; }

log "Starting restore benchmark..."
T_START=$(date +%s%3N)   # milliseconds

restic --repo "$REPO" --password-file "$PASSFILE" \
    restore latest --target "$TARGET"

T_END=$(date +%s%3N)
ELAPSED_MS=$(( T_END - T_START ))
ELAPSED_S=$(echo "scale=1; $ELAPSED_MS / 1000" | bc)

# Measure restored data size
RESTORED_BYTES=$(du -sb "$TARGET" | cut -f1)
RESTORED_MB=$(echo "scale=1; $RESTORED_BYTES / 1048576" | bc)
THROUGHPUT_MB=$(echo "scale=1; $RESTORED_MB / ($ELAPSED_S)" | bc)

log "=== Benchmark Results ==="
log "Restored data:    ${RESTORED_MB} MB"
log "Elapsed time:     ${ELAPSED_S} seconds"
log "Throughput:       ${THROUGHPUT_MB} MB/s"
log "Snapshot count:   $(restic --repo "$REPO" --password-file "$PASSFILE" snapshots --json | python3 -c 'import sys,json;print(len(json.load(sys.stdin)))')"

# Record result
echo "$(date -Iseconds) restored_mb=${RESTORED_MB} elapsed_s=${ELAPSED_S} throughput_mb_s=${THROUGHPUT_MB}" \
    >> /var/log/backup/benchmark.log

log "Results appended to /var/log/backup/benchmark.log"
```

### 6.2 Throughput reference targets

| Storage type | Expected restore throughput |
|-------------|---------------------------|
| Local NVMe | 500–3000 MB/s |
| Local SATA SSD | 200–500 MB/s |
| Local HDD | 80–160 MB/s |
| Gigabit LAN (SFTP) | 50–100 MB/s |
| AWS S3 (restic) | 20–100 MB/s (network dependent) |
| MinIO (LAN) | 80–200 MB/s |

If measured throughput is significantly below these ranges, investigate:
- Repository fragmentation: `restic check` then `restic rebuild-index`
- Network saturation: `iperf3` between backup client and server
- Disk health: `smartctl -t short /dev/sda`

[↑ Table of Contents](#table-of-contents)

---

## 7. Common Restore Failures and Fixes

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| `tar: Cannot change ownership` | Restoring as non-root | Run as root or use `--no-same-owner` |
| SELinux denials after restore | Labels not preserved | `restorecon -Rv /restore/path` |
| `xfsrestore: your filesystem has duplicate UUIDs` | Cloned XFS without remount | `mount -o nouuid /dev/vg/lv /mnt/point` |
| `restic: wrong password or no key found` | Password mismatch | Verify `/etc/backup/restic-password` content |
| Files restored but service fails to start | Config file permissions wrong | `chmod` and `chown` to match originals |
| Incremental tar restore missing files | Restored in wrong order | Reapply full first, then each incremental in date order |
| `grub2: no such device` after bare-metal | UUID changed after reformat | Update `/etc/fstab` and rerun `grub2-mkconfig` |
| MariaDB won't start after restore | InnoDB log file mismatch | Remove `ib_logfile*`, let MariaDB recreate on start |

[↑ Table of Contents](#table-of-contents)

---

## Lab 10 — Restore Drills

### Prerequisites

- Modules 02–08 completed
- At least one backup created by each tool
- Staging directory: `/restore/staging` (create it)

```bash
[server]# mkdir -p /restore/staging
```

[↑ Table of Contents](#table-of-contents)

---

### Lab Part A — tar Restore Drill

**Step 1:** Create a test backup.

```bash
[server]# tar -czf /mnt/backup/lab10-etc.tar.gz \
    --xattrs --xattrs-include='*' --selinux --acls \
    /etc/hostname /etc/fstab /etc/passwd /etc/hosts
```

**Step 2:** Verify archive integrity.

```bash
[server]# tar -tzf /mnt/backup/lab10-etc.tar.gz
```

**Step 3:** Restore to staging.

```bash
[server]# tar -xzf /mnt/backup/lab10-etc.tar.gz \
    --xattrs --xattrs-include='*' --selinux --acls \
    -C /restore/staging/
```

**Step 4:** Verify restored files.

```bash
[server]# ls -lZ /restore/staging/etc/
diff /etc/hostname /restore/staging/etc/hostname && echo "hostname: OK"
diff /etc/fstab    /restore/staging/etc/fstab    && echo "fstab: OK"
grep -q "^root:" /restore/staging/etc/passwd     && echo "passwd: OK"
```

[↑ Table of Contents](#table-of-contents)

---

### Lab Part B — Restic Restore Drill

**Step 1:** Restore the latest snapshot to staging.

```bash
[server]# restic --repo /mnt/backup/restic \
    --password-file /etc/backup/restic-password \
    restore latest \
    --include /etc/hostname \
    --include /etc/fstab \
    --target /restore/staging/restic/
```

**Step 2:** Verify the restored files.

```bash
[server]# diff /etc/fstab /restore/staging/restic/etc/fstab && echo "fstab: OK"
```

**Step 3:** Browse snapshot as FUSE mount (if `fuse` package installed).

```bash
[server]# dnf install -y fuse
mkdir -p /mnt/restic_browse

restic --repo /mnt/backup/restic \
    --password-file /etc/backup/restic-password \
    mount /mnt/restic_browse &
RESTIC_PID=$!

ls /mnt/restic_browse/snapshots/
ls /mnt/restic_browse/snapshots/latest/etc/

# Unmount
fusermount -u /mnt/restic_browse
wait $RESTIC_PID 2>/dev/null || true
```

[↑ Table of Contents](#table-of-contents)

---

### Lab Part C — Automated Verification Script

**Step 1:** Deploy and run the verification script.

```bash
[server]# cat > /usr/local/sbin/verify-backup.sh << 'SCRIPTEOF'
#!/usr/bin/env bash
set -euo pipefail
RESTIC_REPO="${RESTIC_REPO:-/mnt/backup/restic}"
RESTIC_PASSWORD_FILE="${RESTIC_PASSWORD_FILE:-/etc/backup/restic-password}"
VERIFY_TARGET="/tmp/backup-verify-$$"
mkdir -p "$VERIFY_TARGET"
trap 'rm -rf "$VERIFY_TARGET"' EXIT

echo "=== Repository check ==="
restic --repo "$RESTIC_REPO" --password-file "$RESTIC_PASSWORD_FILE" check

echo "=== Restore critical files ==="
restic --repo "$RESTIC_REPO" --password-file "$RESTIC_PASSWORD_FILE" \
    restore latest \
    --include /etc/passwd --include /etc/fstab --include /etc/hostname \
    --target "$VERIFY_TARGET"

echo "=== Verify contents ==="
for f in etc/passwd etc/fstab etc/hostname; do
    [[ -s "${VERIFY_TARGET}/${f}" ]] && echo "OK: $f" || { echo "FAIL: $f missing"; exit 1; }
done

grep -q '^root:' "${VERIFY_TARGET}/etc/passwd" && echo "OK: root in passwd" || { echo "FAIL: no root in passwd"; exit 1; }
echo "=== Verification PASSED ==="
SCRIPTEOF
chmod 700 /usr/local/sbin/verify-backup.sh
```

**Step 2:** Run it.

```bash
[server]# /usr/local/sbin/verify-backup.sh
```

[↑ Table of Contents](#table-of-contents)

---

### Lab Part D — Restore Benchmark

**Step 1:** Run the benchmark and record throughput.

```bash
[server]# T_START=$(date +%s)

restic --repo /mnt/backup/restic \
    --password-file /etc/backup/restic-password \
    restore latest \
    --target /tmp/restore-bench-$$

T_END=$(date +%s)
echo "Elapsed: $(( T_END - T_START )) seconds"
du -sh /tmp/restore-bench-*/
rm -rf /tmp/restore-bench-*
```

**Step 2:** Compare to the reference table in Section 6.2.

[↑ Table of Contents](#table-of-contents)

---

### Lab Part E — Smoke Test

**Step 1:** Run the smoke test on the current live system (should all pass).

```bash
[server]# cat > /usr/local/sbin/smoke-test.sh << 'SMOKEEOF'
#!/usr/bin/env bash
PASS=0; FAIL=0
ok()   { echo "[PASS] $*"; ((PASS++)); }
fail() { echo "[FAIL] $*"; ((FAIL++)); }

[[ "$(hostname)" != "" ]] && ok "hostname set" || fail "hostname empty"
systemctl is-system-running &>/dev/null && ok "systemd running" || fail "systemd degraded"
mountpoint -q / && ok "/ mounted" || fail "/ not mounted"
sestatus | grep -q "enforcing" && ok "SELinux enforcing" || fail "SELinux not enforcing"
for svc in sshd chronyd rsyslog; do
    systemctl is-active --quiet "$svc" && ok "$svc active" || fail "$svc not active"
done
getent passwd root &>/dev/null && ok "root exists" || fail "root missing"
echo ""
echo "Results: ${PASS} passed, ${FAIL} failed"
[[ "$FAIL" -eq 0 ]] && exit 0 || exit 1
SMOKEEOF
chmod 700 /usr/local/sbin/smoke-test.sh
/usr/local/sbin/smoke-test.sh
```

[↑ Table of Contents](#table-of-contents)

---

## Review Questions

1. List five common reasons a backup can exist but a restore can fail.
2. What `restic` command allows you to browse a snapshot interactively without extracting files?
3. In bare-metal recovery, why must `/etc/fstab` UUIDs be updated after restoring to new hardware?
4. What does `.autorelabel` in the root filesystem cause during the next boot?
5. What is the purpose of the `--read-data-subset=10%` flag in `restic check`?
6. Why should you always restore to a staging directory before touching live data?
7. What two commands rebuild the bootloader during a bare-metal restore?
8. Name three metrics that should be recorded during a DR drill to assess RTO.

[↑ Table of Contents](#table-of-contents)

---

## Answers to Review Questions

1. **Five common restore failure reasons:**
   1. Archive corruption (bit rot, incomplete write)
   2. Lost or wrong encryption password/key
   3. Missing backup scope (critical path not included in backup)
   4. Wrong restore order for incremental chain
   5. SELinux label loss causing services to fail even with correct files

2. `restic mount /mnt/restic_browse` — mounts the repository as a FUSE filesystem, allowing browsing with standard tools (`ls`, `cp`, `find`) without extracting.

3. When a disk is reformatted or restored to different hardware, the partition UUIDs change. If `/etc/fstab` still references old UUIDs, the system cannot find its filesystems at boot, causing a boot failure or emergency shell.

4. Touching `/.autorelabel` triggers a full SELinux filesystem relabelling pass on the next boot. This is necessary after bare-metal restore because file context labels stored as xattrs may be absent or incorrect on newly-formatted filesystems.

5. `--read-data-subset=10%` tells restic to read and verify 10% of the stored pack files rather than all of them. This is a balance: a full read is thorough but slow; the subset check catches most corruption while completing quickly enough to run regularly.

6. Restoring to staging first allows you to verify correctness, check permissions, and compare against expected output without risking damage to live production data. If the restore is wrong or incomplete, production continues unaffected.

7. `grub2-install /dev/sda` (installs the bootloader to the MBR/EFI partition) and `grub2-mkconfig -o /boot/grub2/grub.cfg` (regenerates the GRUB configuration file with correct kernel paths and UUIDs).

8. **Three RTO metrics from a DR drill:**
   - Time from declared incident to first restore command (T₀ → T_start)
   - Time from restore start to OS boot (T_start → T_boot)
   - Time from OS boot to all smoke tests passing (T_boot → T_ok)
   Total RTO = T_ok − T₀

© 2026 Jaco Steyn — Licensed under CC BY-SA 4.0
