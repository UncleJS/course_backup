# Module 01 — RHEL 10 Filesystem & What to Back Up

## Learning Objectives

By the end of this module you will be able to:

- Describe the Filesystem Hierarchy Standard (FHS) as implemented in RHEL 10
- Identify which directories must be backed up and why
- Identify which directories should be excluded from backups and why
- Understand RHEL 10-specific paths: systemd units, DNF repos, SELinux policy, podman storage
- Build a prioritised backup inclusion and exclusion list for a typical RHEL 10 server

---

## Table of Contents

- [1. The Filesystem Hierarchy Standard (FHS)](#1-the-filesystem-hierarchy-standard-fhs)
  - [RHEL 10 Specific Notes](#rhel-10-specific-notes)
- [2. Directory-by-Directory Reference](#2-directory-by-directory-reference)
  - [2.1 `/etc` — System Configuration](#21-etc--system-configuration)
  - [2.2 `/home` — User Home Directories](#22-home--user-home-directories)
  - [2.3 `/root` — Root Home Directory](#23-root--root-home-directory)
  - [2.4 `/var` — Variable Data](#24-var--variable-data)
  - [2.5 `/boot` — Boot Files](#25-boot--boot-files)
  - [2.6 `/srv` — Service Data](#26-srv--service-data)
  - [2.7 `/opt` — Optional Software](#27-opt--optional-software)
  - [2.8 `/usr` — Installed Programs & Libraries](#28-usr--installed-programs--libraries)
  - [2.9 `/tmp` and `/var/tmp`](#29-tmp-and-vartmp)
  - [2.10 Paths to ALWAYS EXCLUDE](#210-paths-to-always-exclude)
  - [2.11 RHEL 10 Specific: Podman Container Storage](#211-rhel-10-specific-podman-container-storage)
  - [2.12 RHEL 10 Specific: SELinux File Contexts](#212-rhel-10-specific-selinux-file-contexts)
- [3. Backup Inclusion & Exclusion Reference Table](#3-backup-inclusion--exclusion-reference-table)
  - [Must Include](#must-include)
  - [Usually Include](#usually-include)
  - [Exclude](#exclude)
- [4. Calculating Backup Size](#4-calculating-backup-size)
- [5. Special Considerations by Server Role](#5-special-considerations-by-server-role)
  - [Web Server (Apache / Nginx)](#web-server-apache--nginx)
  - [Database Server (MariaDB)](#database-server-mariadb)
  - [Mail Server (Postfix / Dovecot)](#mail-server-postfix--dovecot)
  - [KVM Hypervisor](#kvm-hypervisor)
  - [Container Host (Podman)](#container-host-podman)
- [6. Creating Your Backup Manifest](#6-creating-your-backup-manifest)
- [Lab Exercises](#lab-exercises)
  - [Lab 01-1: Audit your system's backup footprint](#lab-01-1-audit-your-systems-backup-footprint)
  - [Lab 01-2: Verify SELinux label preservation](#lab-01-2-verify-selinux-label-preservation)
  - [Lab 01-3: Build your system's backup manifest](#lab-01-3-build-your-systems-backup-manifest)
  - [Lab 01-4: Identify exclusion violations](#lab-01-4-identify-exclusion-violations)
- [Review Questions](#review-questions)
- [Answers to Review Questions](#answers-to-review-questions)

---

## 1. The Filesystem Hierarchy Standard (FHS)

RHEL 10 follows the FHS — a standard layout that defines where different types of files live. Understanding this layout is essential to knowing *what* to back up.

```
/
├── bin  -> usr/bin        (user commands — symlink)
├── boot                   (bootloader, kernel, initramfs)
├── dev                    (device files — virtual, do NOT back up)
├── etc                    (system configuration — MUST back up)
├── home                   (user home directories — MUST back up)
├── lib  -> usr/lib        (libraries — symlink)
├── lib64 -> usr/lib64     (64-bit libraries — symlink)
├── media                  (mount point for removable media)
├── mnt                    (temporary mount points)
├── opt                    (third-party software — usually back up)
├── proc                   (kernel/process virtual FS — do NOT back up)
├── root                   (root user home — MUST back up)
├── run                    (runtime data — do NOT back up)
├── srv                    (service data — usually back up)
├── sys                    (kernel virtual FS — do NOT back up)
├── tmp                    (temporary files — usually exclude)
├── usr                    (programs, libraries — back up selectively)
└── var                    (variable data: logs, databases, mail — MUST back up)
```

### RHEL 10 Specific Notes

- `/bin`, `/lib`, `/lib64`, `/sbin` are **symlinks to `/usr`** equivalents. This is the "UsrMerge" — back up `/usr`, not these symlinks directly.
- Default filesystem is **XFS** (not ext4). This affects which tools you use for filesystem-level dumps.
- **SELinux** is enforcing by default — file context labels are stored as extended attributes and must be preserved in backups.
- **Systemd** manages services; unit files in `/etc/systemd/` are critical config.
- **DNF** module state is stored in `/etc/dnf/` and `/var/lib/dnf/` — relevant for system rebuilds.

---

## 2. Directory-by-Directory Reference

### 2.1 `/etc` — System Configuration

**Priority: CRITICAL**

Everything needed to reconfigure a freshly installed RHEL 10 to match your current system is in `/etc`. This is the single most important directory for system recovery.

| Subdirectory | Contents | Notes |
|--------------|----------|-------|
| `/etc/fstab` | Filesystem mount table | Critical for boot |
| `/etc/passwd`, `/etc/shadow`, `/etc/group` | User/group accounts | Critical |
| `/etc/ssh/` | SSH server config and host keys | Host keys are unique — back them up |
| `/etc/sysconfig/` | Network, selinux, boot settings | RHEL-specific |
| `/etc/systemd/` | Systemd unit overrides | Any custom services |
| `/etc/cron.d/`, `/etc/cron.daily/` etc | Cron jobs | |
| `/etc/dnf/` | DNF configuration, repos | Package manager config |
| `/etc/yum.repos.d/` | Repository definitions | |
| `/etc/selinux/` | SELinux policy config | |
| `/etc/ssl/`, `/etc/pki/` | TLS certificates, CA bundles | |
| `/etc/firewalld/` | Firewall rules | |
| `/etc/NetworkManager/` | Network configuration | |
| `/etc/sudoers`, `/etc/sudoers.d/` | Sudo rules | |
| `/etc/audit/` | Audit daemon config | |
| `/etc/logrotate.d/` | Log rotation config | |
| `/etc/profile.d/` | Shell environment scripts | |

**Size:** Typically 10–50 MB. Back up on every scheduled run.

---

### 2.2 `/home` — User Home Directories

**Priority: HIGH**

All user data: documents, scripts, dotfiles, SSH keys, shell history.

```bash
# See how large /home is
du -sh /home/*
```

**Size:** Varies widely — from MB to TB depending on use.

**Note:** Never exclude hidden files/directories (`.bashrc`, `.ssh/`, `.gnupg/`) — these are often the most important files for a user.

---

### 2.3 `/root` — Root Home Directory

**Priority: CRITICAL**

The root user's home. Contains scripts, SSH keys, `.bashrc`, `.bash_history`, and often critical one-off configuration files placed there by admins.

```bash
# Common important files
/root/.ssh/authorized_keys    # Root SSH access
/root/.ssh/id_*               # Root's SSH keys
/root/bin/                    # Admin scripts
/root/.bashrc, /root/.bash_profile
```

---

### 2.4 `/var` — Variable Data

**Priority: HIGH to CRITICAL** (depends on what is running)

This is where running services store their data. The most important subdirectories:

| Path | Contents | Priority |
|------|----------|----------|
| `/var/lib/mysql/` or `/var/lib/mariadb/` | MariaDB/MySQL data files | CRITICAL (use consistent backup!) |
| `/var/lib/postgresql/` | PostgreSQL data | CRITICAL (use consistent backup!) |
| `/var/lib/pgsql/` | PostgreSQL (RHEL path) | CRITICAL |
| `/var/lib/containers/` | Podman container data | HIGH |
| `/var/lib/libvirt/images/` | KVM VM disk images | HIGH |
| `/var/lib/bareos/` | Bareos catalog data | HIGH (if using Bareos) |
| `/var/lib/dnf/` | DNF history and transaction records | MEDIUM |
| `/var/www/` | Web server content (Apache/Nginx default) | HIGH |
| `/var/mail/` | Local mailboxes | MEDIUM |
| `/var/spool/cron/` | User crontabs | HIGH |
| `/var/log/` | Log files | MEDIUM (retention policy dependent) |
| `/var/named/` | BIND DNS zone data | HIGH (if DNS server) |
| `/var/lib/samba/` | Samba shares and config | HIGH (if file server) |

**⚠️ Database warning:** Never back up live database data files (`/var/lib/mysql/`, `/var/lib/postgresql/`) using file copy methods while the database is running. Use database-native dump tools instead, then back up the dump file. See Module 09 for details.

---

### 2.5 `/boot` — Boot Files

**Priority: HIGH**

Contains the Linux kernel, initramfs (initial RAM filesystem), GRUB bootloader configuration, and EFI files (on UEFI systems).

```
/boot/
├── vmlinuz-*              # Kernel image
├── initramfs-*            # Initial RAM filesystem
├── grub2/                 # GRUB2 configuration
│   ├── grub.cfg           # Main GRUB config (generated)
│   └── grubenv            # GRUB environment variables
└── efi/                   # EFI partition (on UEFI systems)
    └── EFI/
        └── redhat/
```

**Note:** `/boot/efi` is often a separate FAT32 partition. Make sure it is included. On UEFI systems, back up the EFI partition contents explicitly.

**Size:** Usually 500 MB – 2 GB.

---

### 2.6 `/srv` — Service Data

**Priority: HIGH** (application-dependent)

By FHS convention, data served by the system (FTP, HTTP, etc.) lives here. Many sysadmins place application data here.

```bash
# Check what is in /srv
ls -la /srv/
```

---

### 2.7 `/opt` — Optional Software

**Priority: MEDIUM to HIGH**

Third-party and commercial software installs here. If the software stores its own data (config, databases) under `/opt`, it must be backed up.

```bash
# Check for data under /opt
du -sh /opt/*
```

---

### 2.8 `/usr` — Installed Programs & Libraries

**Priority: LOW (usually reproducible)**

Everything under `/usr` can be reinstalled with `dnf reinstall`. However, exceptions exist:

| Path | Notes |
|------|-------|
| `/usr/local/` | Locally compiled/installed software — back up |
| `/usr/local/bin/`, `/usr/local/lib/` | Custom scripts and libraries |
| `/usr/lib/systemd/system/` | Package-installed systemd units (skip — reinstall) |

Generally exclude `/usr` from backups but include `/usr/local/`.

---

### 2.9 `/tmp` and `/var/tmp`

**Priority: EXCLUDE**

Temporary files. Contents are not needed for recovery. `/tmp` is often a `tmpfs` mount and does not survive reboots anyway.

---

### 2.10 Paths to ALWAYS EXCLUDE

These are virtual filesystems, runtime data, or data that would corrupt your backup or waste space:

```
/proc/          # Kernel virtual filesystem — not real files
/sys/           # Kernel sysfs — not real files
/dev/           # Device files — not real files
/run/           # Runtime data (PIDs, sockets) — regenerated at boot
/tmp/           # Temporary files — no recovery value
/var/tmp/       # Temporary files
/var/cache/     # Cached data — reproducible
/var/cache/dnf/ # DNF package cache — can be recreated
/media/         # Removable media mount points
/mnt/           # Temporary mounts
lost+found/     # Filesystem recovery directory — not useful to back up
```

---

### 2.11 RHEL 10 Specific: Podman Container Storage

If the machine runs Podman containers, the storage paths are:

| Path | Contents |
|------|----------|
| `/var/lib/containers/` | System-wide container images and data |
| `~/.local/share/containers/` | Rootless container data (per user) |

**Note:** Container images can be large and are usually rebuildable from a registry. Back up container **volumes** (data), not images unless the images are customised and not stored in a registry.

---

### 2.12 RHEL 10 Specific: SELinux File Contexts

SELinux stores security labels as **extended attributes** on files. When backing up and restoring, these labels must be preserved.

```bash
# View SELinux context on a file
ls -Z /etc/passwd

# Verify extended attributes are preserved
getfattr -d /etc/passwd
```

Tools that preserve xattrs: `tar --xattrs`, `rsync -X`, `cp --preserve=xattr`

If SELinux labels are lost after a restore, use:
```bash
# Relabel entire filesystem on next boot
touch /.autorelabel
reboot
```

---

## 3. Backup Inclusion & Exclusion Reference Table

### Must Include

| Path | Reason |
|------|--------|
| `/etc/` | All system configuration |
| `/root/` | Root admin files, SSH keys |
| `/home/` | User data |
| `/boot/` | Kernel, bootloader |
| `/var/lib/` | Service data (databases, containers) |
| `/var/www/` | Web content |
| `/var/spool/cron/` | User crontabs |
| `/var/named/` | DNS zones (if applicable) |
| `/opt/` | Third-party software data |
| `/srv/` | Application data |
| `/usr/local/` | Locally compiled software |

### Usually Include

| Path | Reason |
|------|--------|
| `/var/log/` | Compliance, audit, forensics |
| `/var/mail/` | Local mail |
| `/var/lib/dnf/` | Package history for rebuilds |

### Exclude

| Path | Reason |
|------|--------|
| `/proc/` | Virtual filesystem |
| `/sys/` | Virtual filesystem |
| `/dev/` | Device files |
| `/run/` | Runtime data |
| `/tmp/` | Temporary files |
| `/var/tmp/` | Temporary files |
| `/var/cache/` | Rebuildable cache |
| `/media/` | Removable media |
| `/mnt/` | Temporary mounts |
| `/backup/` | The backup destination itself! |

---

## 4. Calculating Backup Size

Before setting up any backup system, estimate the data size:

```bash
# Total size of critical directories
du -sh /etc /root /home /boot /var/lib /var/www /opt /srv /usr/local 2>/dev/null

# Quick summary of the whole filesystem excluding virtuals
du -sh --exclude=/proc --exclude=/sys --exclude=/dev --exclude=/run \
        --exclude=/tmp --exclude=/var/tmp --exclude=/var/cache \
        / 2>/dev/null

# Size of individual /var/lib entries (identify the big ones)
du -sh /var/lib/* 2>/dev/null | sort -h

# Find the top 20 largest directories
du -ah / --exclude=/proc --exclude=/sys --exclude=/dev 2>/dev/null \
  | sort -rh | head -20
```

---

## 5. Special Considerations by Server Role

### Web Server (Apache / Nginx)

```
Include:
  /etc/httpd/        or /etc/nginx/
  /var/www/
  /etc/ssl/          (TLS certificates)
  /etc/pki/

Exclude:
  /var/log/httpd/    (optional — logs can be large)
```

### Database Server (MariaDB)

```
Include:
  /etc/my.cnf, /etc/my.cnf.d/
  /var/lib/mysql/    (ONLY if using consistent backup method)
  Database dump files created before backup

Exclude:
  /var/lib/mysql/    (if taking hot file copy without quiescing — unreliable!)
```

### Mail Server (Postfix / Dovecot)

```
Include:
  /etc/postfix/
  /etc/dovecot/
  /var/mail/
  /var/lib/dovecot/
  /home/             (if Maildir format used)
```

### KVM Hypervisor

```
Include:
  /etc/libvirt/
  /var/lib/libvirt/images/    (VM disk images — large!)
  /var/lib/libvirt/           (VM configs and state)

Note: Shut down VMs or use live snapshot for consistency
```

### Container Host (Podman)

```
Include:
  /etc/containers/
  /var/lib/containers/        (or just volumes)
  ~/.local/share/containers/  (rootless, per user)

Recommendation: Back up data volumes, rebuild images from registry
```

---

## 6. Creating Your Backup Manifest

A backup manifest is a text file listing exactly what to include and exclude. You will reference this in every subsequent module.

```bash
# Create the backup manifest
sudo mkdir -p /etc/backup
sudo tee /etc/backup/manifest.conf <<'EOF'
# RHEL 10 Backup Manifest
# Format: one path per line
# Lines starting with + = include
# Lines starting with - = exclude

# === CRITICAL ===
+ /etc/
+ /root/
+ /boot/

# === HIGH PRIORITY ===
+ /home/
+ /var/lib/
+ /var/www/
+ /var/spool/cron/
+ /var/named/
+ /opt/
+ /srv/
+ /usr/local/

# === MEDIUM PRIORITY ===
+ /var/log/
+ /var/mail/

# === EXCLUSIONS ===
- /proc/
- /sys/
- /dev/
- /run/
- /tmp/
- /var/tmp/
- /var/cache/
- /media/
- /mnt/
- /backup/
- /var/lib/containers/storage/overlay/    # Container layers (rebuildable)
EOF
```

---

## Lab Exercises

### Lab 01-1: Audit your system's backup footprint

```bash
# 1. List the size of each must-include directory
for dir in /etc /root /home /boot /var/lib /var/www /opt /srv /usr/local; do
    if [ -d "$dir" ]; then
        size=$(du -sh "$dir" 2>/dev/null | cut -f1)
        echo "$size  $dir"
    fi
done

# 2. Identify directories larger than 1GB in /var/lib
du -sh /var/lib/* 2>/dev/null | awk '$1 ~ /[0-9]G/' | sort -h

# 3. Check if any databases are running that need special handling
systemctl is-active mariadb postgresql 2>/dev/null
```

### Lab 01-2: Verify SELinux label preservation

```bash
# 1. Create a test file with specific SELinux context
sudo touch /tmp/test_selinux.txt
sudo chcon -t httpd_sys_content_t /tmp/test_selinux.txt
ls -Z /tmp/test_selinux.txt

# 2. Copy with and without xattr preservation, compare
sudo cp /tmp/test_selinux.txt /tmp/test_noxattr.txt
sudo cp --preserve=xattr /tmp/test_selinux.txt /tmp/test_withxattr.txt

ls -Z /tmp/test_noxattr.txt    # Context lost (default context applied)
ls -Z /tmp/test_withxattr.txt  # Context preserved
```

### Lab 01-3: Build your system's backup manifest

```bash
# Run the manifest creation command from Section 6 above.
# Then review and customise it for your specific server role.
# Add any application-specific paths that apply to your system.

cat /etc/backup/manifest.conf
```

### Lab 01-4: Identify exclusion violations

```bash
# Check if your backup destination is outside protected paths
# (i.e., not inside /etc, /home, /var — the things you're backing up)
mount | grep backup   # Should show /backup on a separate volume

# Confirm /proc /sys /dev are virtual (not real data)
file /proc /sys /dev
df -T /proc /sys /dev
```

---

## Review Questions

1. What is the "UsrMerge" change in RHEL 10 and what does it mean for backups?
2. Why should you never back up `/proc`, `/sys`, or `/dev`?
3. What is the danger of backing up live MariaDB data files in `/var/lib/mysql/`?
4. Which directory contains all system configuration on RHEL 10?
5. What are SELinux extended attributes and why must they be preserved in backups?
6. Name three RHEL 10-specific paths not found on older distributions that may need to be backed up.
7. Why should the backup destination directory itself be excluded from the backup?
8. What command would you use to check the total size of `/etc` and `/var/lib`?
9. Where are user crontabs stored on RHEL 10?
10. If a server runs Podman containers, what is the recommended backup approach: images or volumes?

---

## Answers to Review Questions

1. **UsrMerge** means `/bin`, `/sbin`, `/lib`, `/lib64` are now symlinks to their `/usr` equivalents. For backups: back up `/usr`, not the old root-level directories (they're just symlinks and don't contain unique data).
2. `/proc`, `/sys`, and `/dev` are **virtual filesystems** generated by the kernel at runtime. They contain no real files on disk. Backing them up wastes space, can cause errors, and restoring them could corrupt a running system.
3. MariaDB keeps data in memory, write-ahead logs, and files simultaneously. A file copy of `/var/lib/mysql/` while the database is running may capture data in an inconsistent state — resulting in a backup that cannot be recovered cleanly.
4. `/etc/` contains all system configuration.
5. SELinux **extended attributes** (`security.selinux`) store the security context label on each file. If labels are lost during restore, SELinux will deny access to files that are mislabelled, breaking applications and services.
6. Any three of: `/var/lib/containers/` (Podman), `/etc/containers/`, `/var/lib/libvirt/images/` (KVM), `/etc/systemd/` overrides, `/etc/dnf/` config, `/var/lib/bareos/` (if using Bareos).
7. If the backup destination is inside the source paths, the backup will include itself, causing infinite loops, inflated archive sizes, and potential corruption.
8. `du -sh /etc /var/lib`
9. User crontabs are in `/var/spool/cron/crontabs/` (one file per user). Root's crontab is at `/var/spool/cron/root`.
10. Back up **volumes** (data), not images. Images can be rebuilt from a container registry. Volumes hold the application data that cannot be recreated.

---

*Previous: [00 — Backup Theory & Fundamentals](00-introduction.md)*
*Next: [02 — tar](02-tar.md)*
