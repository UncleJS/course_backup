# Module 12 — Capstone Lab: Full Disaster Recovery Scenario
[![CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey)](./LICENSE.md)
[![RHEL 10](https://img.shields.io/badge/platform-RHEL%2010-red)](https://access.redhat.com/products/red-hat-enterprise-linux)
[![Backup](https://img.shields.io/badge/Backup-RHEL%2010-blue)](https://access.redhat.com/products/red-hat-enterprise-linux)

## Overview

This capstone lab simulates a realistic disaster: the primary disk on `backup-client` fails catastrophically. You will:

1. Build a complete, multi-tool backup of the running system
2. Simulate total disk failure
3. Restore the system from scratch using three different methods
4. Measure and record Recovery Time Objective (RTO) and Recovery Point Objective (RPO)
5. Run a structured post-mortem and produce a signed DR report

This module integrates every skill from Modules 00–11. It is designed to be run in a VM environment where disk destruction is safe.

[↑ Table of Contents](#table-of-contents)

---

## Lab Environment

```
backup-server   192.168.100.10   RHEL 10   ≥ 40 GB disk   Stores all backups
backup-client   192.168.100.20   RHEL 10   ≥ 20 GB disk   The system that "fails"
```

Both VMs must be running before starting. See `README.md` for initial environment setup.

[↑ Table of Contents](#table-of-contents)

---

## Table of Contents

- [Pre-Lab Checklist](#pre-lab-checklist)
- [Phase 1 — Baseline Documentation](#phase-1--baseline-documentation)
  - [Step 1.1 — Capture system inventory](#step-11--capture-system-inventory)
  - [Step 1.2 — Capture disk layout](#step-12--capture-disk-layout)
  - [Step 1.3 — Capture service state](#step-13--capture-service-state)
  - [Step 1.4 — Create canary files](#step-14--create-canary-files)
- [Phase 2 — Create Comprehensive Backups](#phase-2--create-comprehensive-backups)
  - [Step 2.1 — Initialize backup environment on server](#step-21--initialize-backup-environment-on-server)
  - [Step 2.2 — Full tar backup](#step-22--full-tar-backup)
  - [Step 2.3 — rsync snapshot backup](#step-23--rsync-snapshot-backup)
  - [Step 2.4 — Restic backup](#step-24--restic-backup)
  - [Step 2.5 — xfsdump of data volume (if present)](#step-25--xfsdump-of-data-volume-if-present)
  - [Step 2.6 — Verify all backups are on the server](#step-26--verify-all-backups-are-on-the-server)
  - [Step 2.7 — Copy timing file to server](#step-27--copy-timing-file-to-server)
- [Phase 3 — Simulate Disk Failure](#phase-3--simulate-disk-failure)
  - [Step 3.1 — Pre-destruction verification](#step-31--pre-destruction-verification)
  - [Step 3.2 — Record the disaster timestamp](#step-32--record-the-disaster-timestamp)
  - [Step 3.3 — Simulate disk failure (choose one method)](#step-33--simulate-disk-failure-choose-one-method)
- [Phase 4 — Disaster Recovery Execution](#phase-4--disaster-recovery-execution)
  - [Scenario A — Restore Using Restic (Recommended First Attempt)](#scenario-a--restore-using-restic-recommended-first-attempt)
  - [Scenario B — Restore Using tar (Second Method)](#scenario-b--restore-using-tar-second-method)
  - [Scenario C — Restore Using rsync (Third Method)](#scenario-c--restore-using-rsync-third-method)
- [Phase 5 — Measurement and Analysis](#phase-5--measurement-and-analysis)
  - [Step 5.1 — Compile timing data](#step-51--compile-timing-data)
  - [Step 5.2 — RPO assessment](#step-52--rpo-assessment)
  - [Step 5.3 — RTO comparison table](#step-53--rto-comparison-table)
- [Phase 6 — Post-Mortem Report](#phase-6--post-mortem-report)
- [Phase 7 — Advanced Capstone Extensions](#phase-7--advanced-capstone-extensions)
  - [Extension A — Selective File Restore](#extension-a--selective-file-restore)
  - [Extension B — Point-in-Time Recovery](#extension-b--point-in-time-recovery)
  - [Extension C — Cross-Machine Restore](#extension-c--cross-machine-restore)
  - [Extension D — Automated Recovery Runbook](#extension-d--automated-recovery-runbook)
- [Capstone Completion Criteria](#capstone-completion-criteria)
- [Appendix A — Quick Reference: Restore Commands](#appendix-a--quick-reference-restore-commands)
- [Appendix B — Emergency Contacts Template](#appendix-b--emergency-contacts-template)
- [Appendix C — Course Completion Checklist](#appendix-c--course-completion-checklist)

---

## Pre-Lab Checklist

Before beginning the capstone:

```
[ ] backup-server is running and reachable from backup-client
[ ] backup-client has a second disk /dev/sdb formatted as backupvg (Module 05 setup)
[ ] Restic is installed on both machines (Module 06)
[ ] /etc/backup/restic-password exists on backup-client with chmod 600
[ ] SSH key-based access from backup-client to backup-server is working
[ ] This file is printed or accessible from a machine OTHER than backup-client
[ ] You have at least 90 minutes of uninterrupted time
```

[↑ Table of Contents](#table-of-contents)

---

## Phase 1 — Baseline Documentation

Before any failure simulation, document the current state of the target system. This is your reference for measuring recovery completeness.

### Step 1.1 — Capture system inventory

```bash
[client]# cat > /root/pre-disaster-inventory.txt << 'EOF_INVENTORY'
=== SYSTEM INVENTORY — PRE-DISASTER ===
DATE: $(date -Iseconds)
HOSTNAME: $(hostname -f)
OS: $(cat /etc/redhat-release)
KERNEL: $(uname -r)
UPTIME: $(uptime)
EOF_INVENTORY

# Append live data
echo "DATE: $(date -Iseconds)"               >> /root/pre-disaster-inventory.txt
echo "HOSTNAME: $(hostname -f)"              >> /root/pre-disaster-inventory.txt
echo "OS: $(cat /etc/redhat-release)"       >> /root/pre-disaster-inventory.txt
echo "KERNEL: $(uname -r)"                  >> /root/pre-disaster-inventory.txt
```

### Step 1.2 — Capture disk layout

```bash
[client]# {
    echo "=== DISK LAYOUT ==="
    lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,UUID
    echo ""
    echo "=== LVM LAYOUT ==="
    pvs; vgs; lvs
    echo ""
    echo "=== FSTAB ==="
    cat /etc/fstab
    echo ""
    echo "=== MOUNT POINTS ==="
    findmnt --real
} >> /root/pre-disaster-inventory.txt

cat /root/pre-disaster-inventory.txt
```

### Step 1.3 — Capture service state

```bash
[client]# {
    echo "=== RUNNING SERVICES ==="
    systemctl list-units --type=service --state=running --no-legend \
        | awk '{print $1}'
    echo ""
    echo "=== ENABLED SERVICES ==="
    systemctl list-unit-files --type=service --state=enabled --no-legend \
        | awk '{print $1}'
    echo ""
    echo "=== OPEN PORTS ==="
    ss -tlnp
    echo ""
    echo "=== USER ACCOUNTS ==="
    cut -d: -f1,3,6,7 /etc/passwd | grep -v nologin | grep -v false
} >> /root/pre-disaster-inventory.txt
```

### Step 1.4 — Create canary files

Canary files are test markers you will verify exist in the restored system.

```bash
[client]# # Create canary files in several locations
echo "CANARY-ETC-$(date +%s)"    > /etc/backup-canary.txt
echo "CANARY-HOME-$(date +%s)"   > /home/$(logname)/backup-canary.txt
echo "CANARY-ROOT-$(date +%s)"   > /root/backup-canary.txt

# Record their content
echo "=== CANARY FILES ===" >> /root/pre-disaster-inventory.txt
for f in /etc/backup-canary.txt /home/*/backup-canary.txt /root/backup-canary.txt; do
    echo "$f: $(cat $f 2>/dev/null || echo MISSING)" >> /root/pre-disaster-inventory.txt
done

cat /root/pre-disaster-inventory.txt
```

> **Copy `pre-disaster-inventory.txt` to backup-server before proceeding!**

```bash
[client]# scp /root/pre-disaster-inventory.txt \
    backupuser@192.168.100.10:/backup/pre-disaster-inventory.txt
```

[↑ Table of Contents](#table-of-contents)

---

## Phase 2 — Create Comprehensive Backups

You will create backups using three tools: **tar**, **rsync**, and **restic**. This gives you three independent restore paths to compare.

### Step 2.1 — Initialize backup environment on server

```bash
[server]# mkdir -p /backup/{tar,rsync,restic,xfsdump}
chmod 700 /backup

# Initialize Restic repository
restic init --repo /backup/restic \
    --password-file /etc/backup/restic-password
```

### Step 2.2 — Full tar backup

```bash
[client]# # Record start time
echo "TAR_BACKUP_START=$(date -Iseconds)" >> /root/capstone-times.txt

tar -czpf - \
    --xattrs --xattrs-include='*' --selinux --acls \
    --one-file-system \
    --exclude=/proc \
    --exclude=/sys \
    --exclude=/dev \
    --exclude=/run \
    --exclude=/tmp \
    --exclude=/var/tmp \
    --exclude=/mnt \
    --exclude=/media \
    / \
| ssh backupuser@192.168.100.10 \
    "cat > /backup/tar/client-full-$(date +%Y%m%d_%H%M%S).tar.gz"

echo "TAR_BACKUP_END=$(date -Iseconds)"   >> /root/capstone-times.txt
echo "Tar backup complete."
```

### Step 2.3 — rsync snapshot backup

```bash
[client]# echo "RSYNC_BACKUP_START=$(date -Iseconds)" >> /root/capstone-times.txt

rsync -aAXH \
    --delete \
    --one-file-system \
    --exclude=/proc/ \
    --exclude=/sys/ \
    --exclude=/dev/ \
    --exclude=/run/ \
    --exclude=/tmp/ \
    --exclude=/var/tmp/ \
    --exclude=/mnt/ \
    -e "ssh -i /root/.ssh/backup_ed25519" \
    / \
    backupuser@192.168.100.10:/backup/rsync/client-$(date +%Y%m%d_%H%M%S)/

echo "RSYNC_BACKUP_END=$(date -Iseconds)"  >> /root/capstone-times.txt
echo "rsync backup complete."
```

### Step 2.4 — Restic backup

```bash
[client]# echo "RESTIC_BACKUP_START=$(date -Iseconds)" >> /root/capstone-times.txt

restic backup \
    --repo sftp:backupuser@192.168.100.10:/backup/restic \
    --password-file /etc/backup/restic-password \
    --tag capstone \
    --tag "host:$(hostname -s)" \
    --one-file-system \
    --exclude /proc \
    --exclude /sys \
    --exclude /dev \
    --exclude /run \
    --exclude /tmp \
    --exclude /var/tmp \
    --exclude /mnt \
    /

echo "RESTIC_BACKUP_END=$(date -Iseconds)"  >> /root/capstone-times.txt

# Verify snapshots
restic snapshots \
    --repo sftp:backupuser@192.168.100.10:/backup/restic \
    --password-file /etc/backup/restic-password

echo "Restic backup complete."
```

### Step 2.5 — xfsdump of data volume (if present)

```bash
[client]# if lvs /dev/backupvg/data &>/dev/null && mountpoint -q /mnt/data; then
    echo "XFSDUMP_START=$(date -Iseconds)" >> /root/capstone-times.txt
    xfsdump -l 0 -L "capstone-$(date +%Y%m%d)" -M "capstone-media" \
        -f - /mnt/data \
    | ssh backupuser@192.168.100.10 \
        "cat > /backup/xfsdump/data-level0-$(date +%Y%m%d).dump"
    echo "XFSDUMP_END=$(date -Iseconds)" >> /root/capstone-times.txt
    echo "xfsdump complete."
else
    echo "No /dev/backupvg/data found or not mounted — skipping xfsdump."
fi
```

### Step 2.6 — Verify all backups are on the server

```bash
[server]# echo "=== Backup sizes ==="
du -sh /backup/tar/* /backup/rsync/* 2>/dev/null
du -sh /backup/restic/ 2>/dev/null
du -sh /backup/xfsdump/* 2>/dev/null
echo ""
echo "=== Restic snapshots ==="
restic snapshots \
    --repo /backup/restic \
    --password-file /etc/backup/restic-password
```

### Step 2.7 — Copy timing file to server

```bash
[client]# scp /root/capstone-times.txt \
    backupuser@192.168.100.10:/backup/capstone-times.txt
scp /root/pre-disaster-inventory.txt \
    backupuser@192.168.100.10:/backup/pre-disaster-inventory.txt
```

[↑ Table of Contents](#table-of-contents)

---

## Phase 3 — Simulate Disk Failure

> ⚠️ **WARNING: The following steps destroy data on backup-client. Only proceed in a VM or expendable machine. Ensure Phase 2 is fully complete and you have confirmed backups on backup-server.**

### Step 3.1 — Pre-destruction verification

```bash
[server]# # Confirm all three backup types exist before destroying the client
ls -lh /backup/tar/
ls -lh /backup/rsync/
restic snapshots --repo /backup/restic --password-file /etc/backup/restic-password
echo "All backups confirmed on server. Proceed with destruction."
```

### Step 3.2 — Record the disaster timestamp

```bash
[server]# echo "DISASTER_TIME=$(date -Iseconds)" >> /backup/capstone-times.txt
echo "=== DISASTER SIMULATED ==="
```

### Step 3.3 — Simulate disk failure (choose one method)

**Method A — VM snapshot rollback (safest)**
```
In your hypervisor (KVM/libvirt, VMware, VirtualBox):
1. Revert the backup-client VM to a clean RHEL 10 install snapshot
   (before any data was created)
   OR
2. Delete and recreate the VM from scratch with the same IP/hostname
```

**Method B — Wipe the disk (destructive, most realistic)**
```bash
[client]# # ⚠️ THIS DESTROYS ALL DATA ON /dev/sda ⚠️
# ONLY run this if you are certain this is a dispensable VM
# and have confirmed your backups on backup-server in Step 3.1

# Overwrite the partition table and first sectors
dd if=/dev/zero of=/dev/sda bs=1M count=10
sync
# Reboot into rescue media (system will no longer boot normally)
reboot
```

**Method C — LVM thin snapshot "freeze" simulation (non-destructive alternative)**
```bash
# If you want to practice restore procedures without actual destruction:
# Create a new VM from the same RHEL 10 base image with the same IP
# and treat it as the "recovered" machine. This simulates a new-hardware
# scenario rather than a repaired disk, which is actually more realistic.
```

[↑ Table of Contents](#table-of-contents)

---

## Phase 4 — Disaster Recovery Execution

> You are now working on a machine with NO operating system (or a clean OS). Record the start time.

```bash
# On backup-server or a separate terminal:
echo "RECOVERY_START=$(date -Iseconds)" >> /backup/capstone-times.txt
```

[↑ Table of Contents](#table-of-contents)

---

### Scenario A — Restore Using Restic (Recommended First Attempt)

This is the fastest path for a running RHEL 10 rescue environment.

#### Step A1 — Boot rescue environment

Boot the `backup-client` VM from RHEL 10 install ISO.
- Select: **Troubleshooting → Rescue a Red Hat Enterprise Linux system**
- Select: **Skip to shell**

```bash
# You are now in a minimal rescue shell
ip link set eth0 up
dhclient eth0    # or: ip addr add 192.168.100.20/24 dev eth0
ping 192.168.100.10 && echo "Network: OK"
```

#### Step A2 — Install restic in rescue environment

```bash
# Download statically-linked restic binary
curl -Lo /usr/local/bin/restic \
    https://github.com/restic/restic/releases/latest/download/restic_linux_amd64.bz2
# If bz2:
bzip2 -d /usr/local/bin/restic.bz2 2>/dev/null || true
curl -Lo /usr/local/bin/restic \
    https://github.com/restic/restic/releases/latest/download/restic_linux_amd64
chmod +x /usr/local/bin/restic

# Verify
/usr/local/bin/restic version
```

#### Step A3 — Recreate partitions and filesystems

Refer to your pre-disaster-inventory.txt for the original disk layout.

```bash
# Fetch inventory from backup server
scp backupuser@192.168.100.10:/backup/pre-disaster-inventory.txt /tmp/

# Re-partition (adjust sizes to match your original layout)
fdisk /dev/sda << 'FDISK'
g
n
1

+512M
t
1
uefi
n
2


w
FDISK

# Format
mkfs.vfat -F32 /dev/sda1    # EFI
mkfs.xfs  -f   /dev/sda2    # root

# Mount
mount /dev/sda2 /mnt/sysroot
mkdir -p /mnt/sysroot/boot/efi
mount /dev/sda1 /mnt/sysroot/boot/efi
```

#### Step A4 — Restore from restic

```bash
echo "RESTIC_RESTORE_START=$(date -Iseconds)"

export RESTIC_PASSWORD_FILE=/tmp/restic-password
# Get password from backup server
scp backupuser@192.168.100.10:/etc/backup/restic-password /tmp/restic-password

/usr/local/bin/restic restore latest \
    --repo sftp:backupuser@192.168.100.10:/backup/restic \
    --password-file /tmp/restic-password \
    --target /mnt/sysroot/

echo "RESTIC_RESTORE_END=$(date -Iseconds)"
echo "Restic restore complete. Files restored:"
ls /mnt/sysroot/
```

#### Step A5 — Rebuild bootloader

```bash
# Bind-mount virtual filesystems
for d in dev proc sys run; do
    mount --bind /$d /mnt/sysroot/$d
done
mount --bind /sys/firmware/efi/efivars /mnt/sysroot/sys/firmware/efi/efivars 2>/dev/null || true

# Chroot
chroot /mnt/sysroot /bin/bash << 'CHROOT'
# Update fstab UUIDs if disk was reformatted
blkid
# Edit /etc/fstab if necessary to match new UUIDs from blkid output

# Reinstall GRUB for UEFI
dnf reinstall -y grub2-efi grub2-tools shim 2>/dev/null || \
    grub2-install --target=x86_64-efi --efi-directory=/boot/efi

grub2-mkconfig -o /boot/grub2/grub.cfg

# Rebuild initramfs
dracut -f --regenerate-all

# Schedule SELinux relabelling
touch /.autorelabel

echo "Bootloader rebuilt"
CHROOT
```

#### Step A6 — Reboot and validate

```bash
# Unmount
sync
umount -R /mnt/sysroot
reboot
```

After first boot (SELinux relabelling takes 5–15 min):

```bash
[client]# echo "RESTIC_SYSTEM_UP=$(date -Iseconds)"

# Run smoke tests
/usr/local/sbin/smoke-test.sh

# Verify canary files
for f in /etc/backup-canary.txt /root/backup-canary.txt; do
    [[ -f "$f" ]] && echo "FOUND: $f: $(cat $f)" || echo "MISSING: $f"
done

# Compare inventory
diff <(cat /root/pre-disaster-inventory.txt | grep CANARY) \
     <(for f in /etc/backup-canary.txt /root/backup-canary.txt; do
           echo "$f: $(cat $f 2>/dev/null || echo MISSING)"
       done)
```

[↑ Table of Contents](#table-of-contents)

---

### Scenario B — Restore Using tar (Second Method)

Repeat Phase 3 destruction (or use a second test VM), then:

#### Step B1 — Boot rescue, recreate disk layout (same as A1–A3)

#### Step B2 — Find the tar archive on backup-server

```bash
[client-rescue]# ssh backupuser@192.168.100.10 "ls -lh /backup/tar/"
# Note the filename of the most recent archive
```

#### Step B3 — Restore from tar over SSH

```bash
echo "TAR_RESTORE_START=$(date -Iseconds)"

ssh backupuser@192.168.100.10 \
    "cat /backup/tar/client-full-YYYYMMDD_HHMMSS.tar.gz" \
| tar -xzp \
      --xattrs --xattrs-include='*' --selinux --acls \
      --numeric-owner \
      -C /mnt/sysroot/

echo "TAR_RESTORE_END=$(date -Iseconds)"
ls /mnt/sysroot/
```

#### Step B4 — Rebuild bootloader (same as Step A5)

#### Step B5 — Reboot and validate (same as Step A6)

[↑ Table of Contents](#table-of-contents)

---

### Scenario C — Restore Using rsync (Third Method)

#### Step C1 — Boot rescue, recreate disk layout (same as A1–A3)

#### Step C2 — rsync from snapshot back to target

```bash
echo "RSYNC_RESTORE_START=$(date -Iseconds)"

# Find the snapshot on the server
ssh backupuser@192.168.100.10 "ls /backup/rsync/"
SNAPSHOT="client-YYYYMMDD_HHMMSS"   # replace with actual name

rsync -aAXH \
    --numeric-ids \
    -e "ssh -i /root/.ssh/backup_ed25519" \
    backupuser@192.168.100.10:/backup/rsync/${SNAPSHOT}/ \
    /mnt/sysroot/

echo "RSYNC_RESTORE_END=$(date -Iseconds)"
ls /mnt/sysroot/
```

#### Step C3 — Rebuild bootloader and reboot (same as Steps A5–A6)

[↑ Table of Contents](#table-of-contents)

---

## Phase 5 — Measurement and Analysis

After completing at least one (preferably all three) restore scenarios, collect the timing data.

### Step 5.1 — Compile timing data

```bash
[server]# cat /backup/capstone-times.txt

# Calculate durations (example calculation)
python3 << 'PYEOF'
from datetime import datetime

times = {}
with open('/backup/capstone-times.txt') as f:
    for line in f:
        line = line.strip()
        if '=' in line:
            key, val = line.split('=', 1)
            try:
                times[key] = datetime.fromisoformat(val)
            except Exception:
                pass

def diff(a, b):
    if a in times and b in times:
        delta = times[b] - times[a]
        return f"{int(delta.total_seconds())} seconds ({delta.total_seconds()/60:.1f} min)"
    return "N/A"

print("=== Backup durations ===")
print(f"tar backup:    {diff('TAR_BACKUP_START',    'TAR_BACKUP_END')}")
print(f"rsync backup:  {diff('RSYNC_BACKUP_START',  'RSYNC_BACKUP_END')}")
print(f"restic backup: {diff('RESTIC_BACKUP_START', 'RESTIC_BACKUP_END')}")
print()
print("=== Recovery times (RTO per method) ===")
print(f"restic restore:  {diff('RECOVERY_START', 'RESTIC_SYSTEM_UP')}")
print(f"tar restore:     {diff('RECOVERY_START', 'TAR_SYSTEM_UP')}")
print(f"rsync restore:   {diff('RECOVERY_START', 'RSYNC_SYSTEM_UP')}")
PYEOF
```

### Step 5.2 — RPO assessment

```bash
[server]# cat << 'EOF'
RPO Assessment
==============
The backup was created at: [TAR/RSYNC/RESTIC backup end time]
The disaster occurred at:  [DISASTER_TIME]

RPO = DISASTER_TIME - BACKUP_END_TIME

If the disaster happened immediately after backup:
  RPO = approximately 0 (best case)

If the disaster happened just before the next scheduled backup would run:
  RPO = backup interval (e.g., 24 hours for nightly backups)

For this capstone: RPO ≈ _____ minutes (fill in from timing data)
EOF
```

### Step 5.3 — RTO comparison table

Fill in this table with measured results:

| Method | Backup Size | Backup Time | Restore Time | Total RTO | Notes |
|--------|------------|-------------|--------------|-----------|-------|
| tar | | | | | |
| rsync | | | | | |
| restic | | | | | |

[↑ Table of Contents](#table-of-contents)

---

## Phase 6 — Post-Mortem Report

Complete this template. This is a deliverable — treat it as you would in a real incident.

```
=============================================================================
DISASTER RECOVERY POST-MORTEM REPORT
=============================================================================

Incident Reference:  DR-CAPSTONE-[DATE]
Date of Exercise:    [DATE]
Conducted by:        [YOUR NAME]
Reviewed by:         [REVIEWER NAME]
Status:              DRAFT / FINAL

-----------------------------------------------------------------------------
1. EXECUTIVE SUMMARY
-----------------------------------------------------------------------------
A simulated catastrophic disk failure was executed on [backup-client hostname]
at [DISASTER_TIME]. Full system recovery was achieved using [METHOD] in
[TOTAL_RTO] minutes.

Target RTO: _____ minutes
Achieved RTO: _____ minutes
Target RPO: _____ minutes
Achieved RPO: _____ minutes

Result: [ ] MET TARGET   [ ] MISSED TARGET

-----------------------------------------------------------------------------
2. TIMELINE
-----------------------------------------------------------------------------

[TIMESTAMP]  Pre-disaster inventory captured
[TIMESTAMP]  tar backup started
[TIMESTAMP]  tar backup completed (size: _____)
[TIMESTAMP]  rsync backup started
[TIMESTAMP]  rsync backup completed (size: _____)
[TIMESTAMP]  restic backup started
[TIMESTAMP]  restic backup completed (size: _____)
[TIMESTAMP]  DISASTER SIMULATED — disk destroyed
[TIMESTAMP]  Recovery started — booted rescue media
[TIMESTAMP]  Partitions recreated and formatted
[TIMESTAMP]  Restore began (METHOD: _____)
[TIMESTAMP]  Restore completed
[TIMESTAMP]  Bootloader rebuilt
[TIMESTAMP]  System rebooted
[TIMESTAMP]  SELinux relabelling completed
[TIMESTAMP]  Smoke tests PASSED
[TIMESTAMP]  Canary files verified
[TIMESTAMP]  System declared RECOVERED

-----------------------------------------------------------------------------
3. WHAT WORKED WELL
-----------------------------------------------------------------------------
- 
- 
- 

-----------------------------------------------------------------------------
4. WHAT FAILED OR CAUSED DELAYS
-----------------------------------------------------------------------------
- 
- 
- 

-----------------------------------------------------------------------------
5. GAPS IDENTIFIED
-----------------------------------------------------------------------------
Issue: _______________________________________________
Impact: ______________________________________________
Priority: HIGH / MEDIUM / LOW
Action item: _________________________________________
Owner: _______________________________________________
Due date: ____________________________________________

(Repeat for each gap)

-----------------------------------------------------------------------------
6. RTO/RPO ANALYSIS
-----------------------------------------------------------------------------
The primary bottleneck in recovery time was: _______________

Time breakdown:
  Boot rescue media:         _____ min
  Network setup:             _____ min
  Recreate disk layout:      _____ min
  Restore data:              _____ min
  Rebuild bootloader:        _____ min
  Reboot + SELinux relabel:  _____ min
  Validation:                _____ min
  TOTAL:                     _____ min

To reduce RTO below _____ minutes, the following improvements are needed:
1. 
2. 
3. 

-----------------------------------------------------------------------------
7. BACKUP METHOD COMPARISON
-----------------------------------------------------------------------------
Based on this exercise, the recommended primary recovery method is:
  [ ] tar    because: ________________________________
  [ ] rsync  because: ________________________________
  [ ] restic because: ________________________________

Recommended secondary (fallback) method: _________________

-----------------------------------------------------------------------------
8. RUNBOOK UPDATES REQUIRED
-----------------------------------------------------------------------------
The following changes to the DR runbook are required before the next drill:
1. 
2. 
3. 

-----------------------------------------------------------------------------
9. NEXT SCHEDULED DRILL
-----------------------------------------------------------------------------
Date: _______________
Type: [ ] File restore   [ ] Service restore   [ ] Full bare-metal
Assigned to: _______________

-----------------------------------------------------------------------------
10. SIGN-OFF
-----------------------------------------------------------------------------
Conducted by:  ___________________________ Date: _____________
Reviewed by:   ___________________________ Date: _____________
Approved by:   ___________________________ Date: _____________

=============================================================================
END OF REPORT
=============================================================================
```

[↑ Table of Contents](#table-of-contents)

---

## Phase 7 — Advanced Capstone Extensions

If you complete the primary scenario with time to spare, attempt these extensions:

### Extension A — Selective File Restore

Without destroying the system, practice restoring a single deleted file.

```bash
[client]# # Accidentally delete a file
rm /etc/hosts

# Restore just that file from restic
restic restore latest \
    --repo sftp:backupuser@192.168.100.10:/backup/restic \
    --password-file /etc/backup/restic-password \
    --include /etc/hosts \
    --target /

# Verify
cat /etc/hosts
```

### Extension B — Point-in-Time Recovery

Restore from a specific snapshot (not the latest) to simulate recovering to a point before a bad change.

```bash
[client]# # List all snapshots
restic snapshots \
    --repo sftp:backupuser@192.168.100.10:/backup/restic \
    --password-file /etc/backup/restic-password

# Restore from a specific snapshot ID (e.g., abc12345)
restic restore abc12345 \
    --repo sftp:backupuser@192.168.100.10:/backup/restic \
    --password-file /etc/backup/restic-password \
    --include /etc \
    --target /restore/pitr/

diff /etc /restore/pitr/etc/ --brief 2>/dev/null | head -20
```

### Extension C — Cross-Machine Restore

Restore `backup-client`'s `/home` directory to `backup-server` (simulating a migration).

```bash
[server]# restic restore latest \
    --repo /backup/restic \
    --password-file /etc/backup/restic-password \
    --include /home \
    --host backup-client \
    --target /tmp/migrated-home/

ls -la /tmp/migrated-home/home/
```

### Extension D — Automated Recovery Runbook

Write a single script that automates the entire restic bare-metal recovery (suitable for a rescue USB drive).

```bash
[server]# cat > /backup/recovery-runbook.sh << 'RUNBOOK'
#!/usr/bin/env bash
# =============================================================================
# recovery-runbook.sh — Automated bare-metal recovery from restic
# Run from RHEL rescue environment
# =============================================================================
set -euo pipefail

BACKUP_SERVER="192.168.100.10"
BACKUP_USER="backupuser"
RESTIC_REPO="sftp:${BACKUP_USER}@${BACKUP_SERVER}:/backup/restic"
RESTIC_PASSWORD_FILE="/tmp/restic-password"
TARGET_DISK="${1:-/dev/sda}"
SYSROOT="/mnt/sysroot"

echo "=== Bare-metal Recovery Runbook ==="
echo "Target disk: $TARGET_DISK"
echo "Backup server: $BACKUP_SERVER"
echo ""

step() { echo ""; echo ">>> STEP $1: $2"; echo ""; }

step 1 "Network connectivity check"
ping -c2 "$BACKUP_SERVER" || { echo "Cannot reach backup server!"; exit 1; }

step 2 "Get restic password from backup server"
scp "${BACKUP_USER}@${BACKUP_SERVER}:/etc/backup/restic-password" "$RESTIC_PASSWORD_FILE"

step 3 "Install restic"
curl -sSLo /usr/local/bin/restic \
    https://github.com/restic/restic/releases/latest/download/restic_linux_amd64
chmod +x /usr/local/bin/restic
restic version

step 4 "Partition disk $TARGET_DISK"
echo "Manual step: partition the disk using fdisk or sgdisk."
echo "Then run: mkfs.xfs -f ${TARGET_DISK}2 && mount ${TARGET_DISK}2 $SYSROOT"
echo "Press ENTER when disk is formatted and mounted at $SYSROOT..."
read -r

step 5 "Restore from restic"
T_START=$(date +%s)
/usr/local/bin/restic restore latest \
    --repo "$RESTIC_REPO" \
    --password-file "$RESTIC_PASSWORD_FILE" \
    --target "$SYSROOT"
T_END=$(date +%s)
echo "Restore completed in $(( T_END - T_START )) seconds."

step 6 "Bind-mount virtual filesystems"
for d in dev proc sys run; do
    mount --bind /$d ${SYSROOT}/$d
done

step 7 "Rebuild bootloader"
chroot "$SYSROOT" grub2-install "$TARGET_DISK"
chroot "$SYSROOT" grub2-mkconfig -o /boot/grub2/grub.cfg
chroot "$SYSROOT" dracut -f --regenerate-all
touch "${SYSROOT}/.autorelabel"

step 8 "Unmount and reboot"
umount -R "$SYSROOT"
echo "Recovery complete. Rebooting in 10 seconds..."
sleep 10
reboot
RUNBOOK

chmod 755 /backup/recovery-runbook.sh
echo "Recovery runbook saved to /backup/recovery-runbook.sh"
```

[↑ Table of Contents](#table-of-contents)

---

## Capstone Completion Criteria

You have completed the capstone when all of the following are true:

```
[ ] pre-disaster-inventory.txt exists on backup-server
[ ] At least two backup methods completed successfully (tar, rsync, OR restic)
[ ] System was restored from at least ONE backup (bare-metal or VM recreation)
[ ] Smoke tests pass on the restored system
[ ] All canary files are present and match pre-disaster values
[ ] capstone-times.txt has timestamps for backup and recovery phases
[ ] RTO comparison table (Section 5.3) is filled in for at least one method
[ ] Post-mortem report (Phase 6) is completed and signed
[ ] At least one gap or improvement was identified in the post-mortem
[ ] Next drill date is scheduled in the post-mortem
```

[↑ Table of Contents](#table-of-contents)

---

## Appendix A — Quick Reference: Restore Commands

| Tool | Restore command |
|------|----------------|
| tar | `tar -xzpf archive.tar.gz --xattrs --selinux -C /restore/` |
| rsync | `rsync -aAXH snapshot/ /restore/` |
| restic | `restic restore latest --target /restore/` |
| xfsrestore | `xfsrestore -f dump.file /restore/target/` |
| restic (single file) | `restic restore latest --include /path/to/file --target /restore/` |
| restic (FUSE browse) | `restic mount /mnt/browse` |
| Bareos (bconsole) | `restore` wizard in `bconsole` |
| LVM snapshot | `mount -o ro,nouuid /dev/vg/snap /mnt/snap` |

[↑ Table of Contents](#table-of-contents)

---

## Appendix B — Emergency Contacts Template

```
=============================================================================
DR EMERGENCY CONTACTS
=============================================================================
Printed: [DATE]  — Keep this with your DR kit

Role                  Name                Phone            Email
--------------------------------------------------------------------
Primary sysadmin      ________________    ________________ ________________
Secondary sysadmin    ________________    ________________ ________________
Database admin        ________________    ________________ ________________
Network/security      ________________    ________________ ________________
Management contact    ________________    ________________ ________________
Hosting provider      ________________    ________________ ________________
Backup vendor support ________________    ________________ ________________

Backup server IP:      192.168.100.10
Backup server SSH port: 22
Restic password location: /etc/backup/restic-password
  Escrow copy location: ________________________________
GPG key fingerprint:  ________________________________
  Escrow key location: ________________________________

Last DR drill:  ________________
Next DR drill:  ________________
=============================================================================
```

[↑ Table of Contents](#table-of-contents)

---

## Appendix C — Course Completion Checklist

You have mastered the full course when you can independently:

```
FOUNDATIONS (Modules 00–01)
[ ] Define RPO and RTO and explain how they drive backup design
[ ] List and explain the 3-2-1 and 3-2-1-1-0 rules
[ ] Identify which RHEL 10 directories must be backed up and which should be excluded
[ ] Explain why SELinux xattrs must be preserved in backups

TOOLS (Modules 02–08)
[ ] Create full and incremental tar archives with SELinux/ACL preservation
[ ] Use rsync --link-dest to create efficient snapshot trees
[ ] Use xfsdump/xfsrestore for XFS filesystems
[ ] Create, mount, and remove LVM snapshots with database consistency
[ ] Initialize and use a restic repository with local, SFTP, S3, and MinIO backends
[ ] Install and configure Bareos with MariaDB catalog, TLS, and bconsole
[ ] Install and configure Amanda with a VTL for multi-client backups

STRATEGY AND AUTOMATION (Module 09)
[ ] Design a GFS rotation schedule with correct retention tiers
[ ] Write and deploy systemd timer pairs for backup scheduling
[ ] Implement pre/post hooks for database consistency
[ ] Configure email, syslog, and webhook failure alerting

RESTORE TESTING (Module 10)
[ ] Execute per-tool restore procedures from a checklist
[ ] Perform bare-metal recovery including bootloader rebuild
[ ] Run automated restore verification and interpret results
[ ] Conduct a DR drill and measure RTO

SECURITY AND COMPLIANCE (Module 11)
[ ] Encrypt tar archives with GPG and restore from them
[ ] Manage restic repository keys and rotation
[ ] Harden SSH access for backup operations
[ ] Configure auditd rules for backup paths
[ ] Design a compliant offsite strategy
[ ] Apply GDPR/PCI-DSS retention requirements

CAPSTONE (Module 12)
[ ] Execute the full disaster recovery scenario
[ ] Compare restore methods with measured RTO
[ ] Complete a post-mortem report with identified improvements
[ ] Schedule the next DR drill
```

**Congratulations. You now have the knowledge and hands-on experience to design, implement, test, and maintain enterprise-grade backup and disaster recovery systems on RHEL 10.**

© 2026 UncleJS — Licensed under CC BY-NC-SA 4.0
