# RHEL 10 Backup Procedure & Strategy — Complete Course

A comprehensive, zero-to-advanced course on backup procedures, strategies, and tools
for Red Hat Enterprise Linux 10 systems. Each module builds on the previous, progressing
from theory through hands-on implementation to enterprise-grade deployment.

[![CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey)](LICENSE.md)
[![RHEL 10](https://img.shields.io/badge/platform-RHEL%2010-red)](https://access.redhat.com/products/red-hat-enterprise-linux)
[![Backup](https://img.shields.io/badge/Backup-RHEL%2010-blue)](https://access.redhat.com/products/red-hat-enterprise-linux)

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Course Structure](#course-structure)
- [Recommended Learning Paths](#recommended-learning-paths)
- [Lab Environment Setup](#lab-environment-setup)
- [Conventions Used in This Course](#conventions-used-in-this-course)
- [Tools Covered](#tools-covered)
- [Key Concepts Quick Reference](#key-concepts-quick-reference)
- [License](#license)

---

## Prerequisites

| Requirement | Detail |
|-------------|--------|
| OS | RHEL 10 (or compatible: AlmaLinux 10, Rocky Linux 10) |
| Access | Root or sudo privileges |
| Networking | Two machines recommended for client/server labs (VMs acceptable) |
| Storage | A second disk or LVM volume for practice (`/dev/sdb` used in the lab setup below) |
| Packages | `dnf` access (local mirror or internet) |

---

## Course Structure

| Module | File | Topic | Level |
|--------|------|--------|-------|
| 00 | [00-introduction.md](00-introduction.md) | Backup Theory & Fundamentals | Beginner |
| 01 | [01-rhel10-filesystem.md](01-rhel10-filesystem.md) | RHEL 10 Filesystem & What to Back Up | Beginner |
| 02 | [02-tar.md](02-tar.md) | `tar` — Archives, Compression & Incrementals | Beginner–Intermediate |
| 03 | [03-rsync.md](03-rsync.md) | `rsync` — Local/Remote Sync & Snapshot-Style Incrementals | Intermediate |
| 04 | [04-dump-restore.md](04-dump-restore.md) | `dump` / `xfsdump` — Filesystem-Level Backup | Intermediate |
| 05 | [05-lvm-snapshots.md](05-lvm-snapshots.md) | LVM Snapshots — Consistent Pre-Backup Freezes | Intermediate |
| 06 | [06-restic.md](06-restic.md) | Restic — Modern Deduplicated Encrypted Backups | Intermediate–Advanced |
| 07 | [07-bareos.md](07-bareos.md) | Bareos — Enterprise Backup (Full Install & Deep Configuration) | Advanced |
| 08 | [08-amanda.md](08-amanda.md) | Amanda — Network Backup Server | Advanced |
| 09 | [09-strategy-automation.md](09-strategy-automation.md) | Backup Strategy and Automation | Intermediate–Advanced |
| 10 | [10-restore-testing.md](10-restore-testing.md) | Restore Testing and Disaster Recovery Drills | All Levels |
| 11 | [11-security-compliance.md](11-security-compliance.md) | Backup Security and Compliance | Advanced |
| 12 | [12-capstone-lab.md](12-capstone-lab.md) | Capstone Lab: Full Disaster Recovery Scenario | Advanced |

---

## Recommended Learning Paths

### Path A — New to Backups
Follow modules in order: 00 → 01 → 02 → 03 → 09 → 10

### Path B — Intermediate Sysadmin
00 → 01 → 03 → 05 → 06 → 09 → 10 → 11

### Path C — Enterprise / Production Focus
00 → 01 → 05 → 07 → 09 → 10 → 11 → 12

### Path D — Full Course (Zero to Advanced)
00 → 01 → 02 → 03 → 04 → 05 → 06 → 07 → 08 → 09 → 10 → 11 → 12

---

## Lab Environment Setup

All labs use a consistent test environment. Set this up before starting Module 02.

### Minimum setup (single machine)

```bash
# Create a second VG and LV for practice
# Assumes /dev/sdb is available (add a 20GB disk to your VM)
sudo pvcreate /dev/sdb
sudo vgcreate backupvg /dev/sdb
sudo lvcreate -L 10G -n datalv backupvg
sudo mkfs.xfs /dev/backupvg/datalv
sudo mkdir -p /data
sudo mount /dev/backupvg/datalv /data

# Create a backup destination volume
sudo lvcreate -L 8G -n backuplv backupvg
sudo mkfs.xfs /dev/backupvg/backuplv
sudo mkdir -p /backup
sudo mount /dev/backupvg/backuplv /backup

# Populate /data with test content
sudo mkdir -p /data/{configs,logs,documents,databases}
sudo cp -r /etc/. /data/configs/
sudo dd if=/dev/urandom bs=1M count=50 | sudo tee /data/documents/testfile.bin > /dev/null
echo "Lab environment ready."
```

### Recommended: two-machine setup

| Role | Hostname | IP Example |
|------|----------|------------|
| Backup Server | `backup-server` | 192.168.100.10 |
| Client Machine | `backup-client` | 192.168.100.20 |

Add these to `/etc/hosts` on both machines:
```
192.168.100.10  backup-server
192.168.100.20  backup-client
```

---

## Conventions Used in This Course

- `#` prompt = command runs as **root**
- `$` prompt = command runs as **regular user**
- `[server]#` = run on backup-server
- `[client]#` = run on backup-client
- Config blocks show **every directive** with inline comments (`# explanation`)
- Each module ends with **Lab Exercises** and **Review Questions**
- Answers to Review Questions are provided at the bottom of each module
- Exception: Module 12 is a hands-on capstone structured as phases with completion checklists — it has no Review Questions section by design

---

## Tools Covered

| Tool | Type | Use Case |
|------|------|----------|
| `tar` | Archive | Simple file/directory archives, incrementals |
| `rsync` | Sync | Efficient local/remote sync, snapshot-style incrementals |
| `dump` / `xfsdump` | Filesystem | Level-based filesystem dumps (XFS native) |
| LVM Snapshots | Block | Crash-consistent point-in-time snapshots |
| Restic | Dedup+Encrypt | Modern, encrypted, deduplicated backups to any backend |
| Bareos | Enterprise | Full client/server backup with catalog, scheduling, pools |
| Amanda | Enterprise | Network backup server with tape/disk VTL support |

---

## Key Concepts Quick Reference

| Term | Definition |
|------|------------|
| RPO | Recovery Point Objective — maximum acceptable data loss (time) |
| RTO | Recovery Time Objective — maximum acceptable downtime |
| Full Backup | Complete copy of all selected data |
| Incremental | Only changes since the last backup (of any kind) |
| Differential | Only changes since the last **full** backup |
| 3-2-1 Rule | 3 copies, 2 media types, 1 offsite |
| Deduplication | Storing identical data blocks only once |
| Retention Policy | Rules defining how long backups are kept |
| Catalog | Database of backup metadata (what was backed up, when, where) |

## License

This project is licensed under the
Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International (CC BY-NC-SA 4.0).

https://creativecommons.org/licenses/by-nc-sa/4.0/

---

*Course version: 1.0 — RHEL 10 — February 2026 — All 13 modules complete*


© 2026 UncleJS — Licensed under CC BY-NC-SA 4.0
