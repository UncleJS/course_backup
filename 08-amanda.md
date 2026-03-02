# Module 08 — Amanda — Network Backup Server
[![CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey)](./LICENSE.md)
[![RHEL 10](https://img.shields.io/badge/platform-RHEL%2010-red)](https://access.redhat.com/products/red-hat-enterprise-linux)
[![Backup](https://img.shields.io/badge/Backup-RHEL%2010-blue)](https://access.redhat.com/products/red-hat-enterprise-linux)

## Learning Objectives

By the end of this module you will be able to:

- Understand Amanda's architecture and concepts
- Install and configure Amanda server and client on RHEL 10
- Configure `amanda.conf` with all major directives explained
- Define `disklist` entries for all clients
- Run `amcheck`, `amdump`, `amrestore`, and `amadmin`
- Set up a Virtual Tape Library (VTL) on disk
- Automate Amanda with cron
- Restore individual files and full dumps

---

## Table of Contents

- [1. What is Amanda?](#1-what-is-amanda)
  - [Key characteristics](#key-characteristics)
  - [Amanda vs Bareos](#amanda-vs-bareos)
- [2. Architecture](#2-architecture)
- [3. Installation](#3-installation)
  - [3.1 Install Amanda server](#31-install-amanda-server)
  - [3.2 Create the Amanda user](#32-create-the-amanda-user)
  - [3.3 Set up SSH key authentication](#33-set-up-ssh-key-authentication)
  - [3.4 Configure .amandahosts on clients](#34-configure-amandahosts-on-clients)
  - [3.5 Install amandad on clients](#35-install-amandad-on-clients)
  - [3.6 Configure xinetd/systemd for amandad (on clients)](#36-configure-xinetdsystemd-for-amandad-on-clients)
- [4. Amanda Configuration Directory](#4-amanda-configuration-directory)
  - [Create a configuration set](#create-a-configuration-set)
- [5. Virtual Tape Library Setup](#5-virtual-tape-library-setup)
  - [Configure the vtape (virtual tape) changer](#configure-the-vtape-virtual-tape-changer)
- [6. amanda.conf — Complete Reference](#6-amandaconf--complete-reference)
  - [Key amanda.conf directives explained](#key-amandaconf-directives-explained)
- [7. disklist — Defining What to Back Up](#7-disklist--defining-what-to-back-up)
  - [disklist with per-DLE options](#disklist-with-per-dle-options)
- [8. Initialise Amanda](#8-initialise-amanda)
- [9. amcheck — Pre-backup Validation](#9-amcheck--pre-backup-validation)
- [10. amdump — Running Backups](#10-amdump--running-backups)
  - [What amdump does](#what-amdump-does)
- [11. Amanda Report](#11-amanda-report)
- [12. amadmin — Administration Tool](#12-amadmin--administration-tool)
- [13. amrestore — Restoring Files](#13-amrestore--restoring-files)
  - [13.1 Find which tape a file is on](#131-find-which-tape-a-file-is-on)
  - [13.2 amrestore — restore from tape/VTL](#132-amrestore--restore-from-tapevtl)
  - [13.3 amrecover — Interactive recovery (recommended)](#133-amrecover--interactive-recovery-recommended)
- [14. Automating Amanda with Cron](#14-automating-amanda-with-cron)
- [15. Multiple Configuration Sets](#15-multiple-configuration-sets)
- [Lab Exercises](#lab-exercises)
  - [Lab 08-1: Install and configure Amanda server](#lab-08-1-install-and-configure-amanda-server)
  - [Lab 08-2: Write amanda.conf and disklist](#lab-08-2-write-amandaconf-and-disklist)
  - [Lab 08-3: Run first backup](#lab-08-3-run-first-backup)
  - [Lab 08-4: Interactive restore with amrecover](#lab-08-4-interactive-restore-with-amrecover)
  - [Lab 08-5: Set up cron automation](#lab-08-5-set-up-cron-automation)
- [Review Questions](#review-questions)
- [Answers to Review Questions](#answers-to-review-questions)

---

## 1. What is Amanda?

Amanda (Advanced Maryland Automatic Network Disk Archiver) is an open-source backup server that can back up multiple clients over a network. Originally designed for tape, it also supports disk-based virtual tape libraries (VTL).

### Key characteristics

| Feature | Description |
|---------|-------------|
| **Tape-first design** | Originally designed for tape changers; disk VTL emulates tape |
| **Automatic scheduling** | Balances full/incremental dumps across the week automatically |
| **Multi-client** | One server backs up many clients over the network |
| **Native compression** | Supports gzip, bzip2 compression on client before transfer |
| **Encryption** | Supports GPG and AES encryption |
| **No agent needed** | Uses `amandad` which is triggered over SSH (no daemon) |

### Amanda vs Bareos

| | Amanda | Bareos |
|--|--------|--------|
| Architecture | Server-centric, SSH-based | Daemons on server and all clients |
| Catalog | Simple text files + index | SQL database (MariaDB) |
| Configuration | Two files (amanda.conf + disklist) | Directory of resource files |
| GUI | AmandaWeb (commercial) | bareos-webui (included) |
| Tape support | Native | Via autochanger |
| Complexity | Medium | High |
| Best for | Simpler networks, tape environments | Large environments, enterprise needs |

[↑ Table of Contents](#table-of-contents)

---

## 2. Architecture

```
┌────────────────────────────────────────────┐
│            AMANDA SERVER                   │
│                                            │
│  amanda.conf      ← Main configuration    │
│  disklist         ← What to back up        │
│  /etc/amanda/     ← Config directory       │
│  /var/lib/amanda/ ← State and indexes      │
│  /backup/vtl/     ← Virtual Tape Library   │
└────────┬───────────────────────────────────┘
         │  SSH
         │
    ┌────▼──────────────┐
    │   amandad          │
    │ (runs on clients)  │
    │  started via SSH   │
    └────────────────────┘
```

Amanda uses SSH to trigger `amandad` on clients. No permanent daemon runs on clients — Amanda connects over SSH, starts `amandad` which performs the backup, and streams data back to the server.

[↑ Table of Contents](#table-of-contents)

---

## 3. Installation

### 3.1 Install Amanda server

```bash
# Enable EPEL for Amanda
sudo dnf install -y epel-release

# Install Amanda server and client packages
sudo dnf install -y amanda amanda-server amanda-client

# Verify
amserverconfig --version 2>/dev/null || amanda --version 2>/dev/null
```

### 3.2 Create the Amanda user

Amanda runs as a dedicated user `amandabackup`:

```bash
# The amanda package creates this user automatically
# Verify:
id amandabackup

# If not created:
sudo useradd -r -m -d /var/lib/amanda -s /bin/bash amandabackup
sudo passwd -l amandabackup   # Lock password — SSH key only
```

### 3.3 Set up SSH key authentication

Amanda uses SSH to connect to clients. Set up key-based auth:

```bash
# On Amanda server: generate key for amandabackup user
sudo -u amandabackup ssh-keygen -t ed25519 \
  -f /var/lib/amanda/.ssh/id_ed25519 \
  -N "" \
  -C "amanda@$(hostname)"

# View the public key
sudo cat /var/lib/amanda/.ssh/id_ed25519.pub

# On each client: create amandabackup user and add key
sudo useradd -r -m -d /var/lib/amanda -s /bin/bash amandabackup
sudo mkdir -p /var/lib/amanda/.ssh
sudo chmod 700 /var/lib/amanda/.ssh
# Copy the server's public key:
echo "ssh-ed25519 AAAA... amanda@backup-server" | \
  sudo tee /var/lib/amanda/.ssh/authorized_keys
sudo chmod 600 /var/lib/amanda/.ssh/authorized_keys
sudo chown -R amandabackup:amandabackup /var/lib/amanda/.ssh
```

### 3.4 Configure .amandahosts on clients

Amanda also uses `.amandahosts` for additional authentication:

```bash
# On each client, create .amandahosts
sudo tee /var/lib/amanda/.amandahosts <<'EOF'
backup-server amandabackup
EOF
sudo chown amandabackup:amandabackup /var/lib/amanda/.amandahosts
sudo chmod 600 /var/lib/amanda/.amandahosts
```

### 3.5 Install amandad on clients

```bash
# On each client:
sudo dnf install -y amanda-client

# Verify amandad is available
which amandad
```

### 3.6 Configure xinetd/systemd for amandad (on clients)

On RHEL 10 systems, Amanda client can be launched via xinetd or directly via SSH:

```bash
# SSH-based approach (recommended, no xinetd needed):
# The server SSH key triggers amandad directly
# Add to client's authorized_keys (replace key):
# command="/usr/libexec/amanda/amandad -auth=ssh amdump amrestore",no-port-forwarding,...

sudo tee /var/lib/amanda/.ssh/authorized_keys <<'EOF'
command="/usr/libexec/amanda/amandad -auth=ssh amdump amrestore",no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty ssh-ed25519 AAAA... amanda@backup-server
EOF
sudo chmod 600 /var/lib/amanda/.ssh/authorized_keys
sudo chown amandabackup:amandabackup /var/lib/amanda/.ssh/authorized_keys
```

[↑ Table of Contents](#table-of-contents)

---

## 4. Amanda Configuration Directory

```
/etc/amanda/
└── DailySet/                # A backup configuration set (one per schedule)
    ├── amanda.conf          # Main Amanda configuration
    ├── disklist             # List of what to back up
    ├── tapelist             # Record of tapes/VTL volumes used
    └── advanced.conf        # (optional) Advanced settings

/var/lib/amanda/
├── DailySet/
│   ├── curinfo/             # Per-DLE statistics (rates, last dump times)
│   └── index/               # Per-client, per-DLE file indexes
└── .ssh/                    # SSH keys for amandabackup
```

### Create a configuration set

```bash
# Create config directory
sudo mkdir -p /etc/amanda/DailySet
sudo chown -R amandabackup:disk /etc/amanda/DailySet

# Create state directories
sudo -u amandabackup mkdir -p /var/lib/amanda/DailySet/{curinfo,index}
```

[↑ Table of Contents](#table-of-contents)

---

## 5. Virtual Tape Library Setup

Amanda traditionally used physical tape. For disk-based storage, we create a VTL — a directory that Amanda treats as a tape changer.

```bash
# Create VTL directory and tape slots
sudo mkdir -p /backup/vtl/DailySet
sudo chown -R amandabackup:disk /backup/vtl/DailySet
sudo chmod 750 /backup/vtl/DailySet

# Create virtual tape slots (10 slots, 10GB each)
sudo -u amandabackup bash <<'EOF'
cd /backup/vtl/DailySet
for i in $(seq -w 01 10); do
  mkdir -p slot${i}
done
# Create the data directory
mkdir -p data
# Create the drive symlink (points to a slot)
ln -sf slot01 drive0
EOF

ls -la /backup/vtl/DailySet/
```

### Configure the vtape (virtual tape) changer

```bash
# Create the chg-disk configuration for virtual tapes
sudo tee /etc/amanda/DailySet/chg-disk.conf <<'EOF'
# chg-disk.conf — Virtual tape changer for disk
tpchanger "chg-disk"
changerfile "/var/lib/amanda/DailySet/changer"
tapedev "file:/backup/vtl/DailySet/drive0"
rawtapedev "/backup/vtl/DailySet"
runtapes 1
EOF
```

[↑ Table of Contents](#table-of-contents)

---

## 6. amanda.conf — Complete Reference

```bash
sudo tee /etc/amanda/DailySet/amanda.conf <<'EOF'
# /etc/amanda/DailySet/amanda.conf
# Amanda Daily Backup Configuration
# Configuration set name
org "DailySet"

# Amanda server hostname
mailto "root@localhost"

# Minimum size required for a dump to run (in KB)
dumpcycle 7            # Aim for a full dump every 7 days
runspercycle 5         # Number of amdump runs per dumpcycle (week)
tapecycle 10           # How many tapes/VTL slots to cycle through
runtapes 1             # Number of tapes to use per run

# How Amanda balances full vs incremental dumps:
# Amanda calculates a "balance" — it automatically promotes DLEs
# to full backup when they haven't had one recently

# Storage paths
infofile "/var/lib/amanda/DailySet/curinfo"    # DLE statistics
logdir   "/var/log/amanda/DailySet"            # Amanda log directory
indexdir "/var/lib/amanda/DailySet/index"      # File indexes (for amrecover)

# Virtual Tape Library
tpchanger "chg-disk:/backup/vtl/DailySet"
tapetype "HARDDISK"
labelstr "^DailySet-[0-9][0-9]*$"              # Regex: valid tape labels

# Tape (VTL slot) definition
define tapetype HARDDISK {
  comment "Disk-based virtual tape"
  length  10 gbytes    # Size of each VTL slot
  filemark  0 kbytes   # No physical filemark for disk
  speed  100 mbytes    # Expected write speed
}

# Dump bandwidth limits (KB/s per client)
netusage 80000       # Total network bandwidth available (80 MB/s)
inparallel 4         # Maximum parallel dump streams

# Compression
# Compression is done on the client before transfer
# compress client fast = use fast gzip on client
# compress client best = use best gzip on client
# compress server fast = compress on server after receive
# compress none = no compression (if data is already compressed)
compress client fast

# Encryption (uses GPG — see Module 11 for key setup)
# encrypt client
# encrypt server

# amandad path on clients
amandad_path "/usr/libexec/amanda/amandad"

# SSH authentication
ssh_keys "/var/lib/amanda/.ssh/id_ed25519"

# Maximum time for a dump to run (seconds)
maxdumps 4

# Reserve space for Amanda's own operations (KB)
reserved-space 100 mbytes

# Hold time before tapes are reused (days)
# Tapes older than this are eligible for overwriting
tapecycle 10

# Dumptypes — define backup profiles
define dumptype global {
  comment "Global defaults"
  compress client fast
  auth "ssh"
  ssh_keys "/var/lib/amanda/.ssh/id_ed25519"
  index yes                  # Create file index (enables amrecover)
  estimate server            # Estimate method
  maxdumps 2
}

define dumptype no-compress {
  global
  comment "No compression (for already-compressed data)"
  compress none
}

define dumptype comp-user {
  global
  comment "Compressed user data"
  compress client fast
  priority medium
}

define dumptype comp-root {
  global
  comment "Compressed system data — higher priority"
  compress client fast
  priority high
}

define dumptype no-index {
  global
  comment "No file index (for large/bulk data)"
  index no
  compress client fast
}

# Interface definitions (for networks)
define interface local {
  comment "Loopback — local dumps"
  maxusage 10000000    # 10 GB/s (essentially unlimited locally)
}

define interface 1Gig {
  comment "Standard gigabit LAN"
  maxusage 80000       # 80 MB/s
}
EOF

# Create the log directory
sudo mkdir -p /var/log/amanda/DailySet
sudo chown amandabackup:disk /var/log/amanda/DailySet
```

### Key amanda.conf directives explained

| Directive | Description |
|-----------|-------------|
| `dumpcycle N` | Target: run a full dump for each DLE every N days |
| `runspercycle N` | How many amdump runs happen per dumpcycle |
| `tapecycle N` | Number of tapes in the cycle before reuse |
| `runtapes N` | Maximum tapes Amanda can use per amdump run |
| `tapetype` | Named tape definition (capacity, speed) |
| `labelstr` | Regex that valid tape labels must match |
| `infofile` | Where Amanda stores DLE statistics |
| `indexdir` | Where Amanda stores file indexes |
| `logdir` | Where Amanda writes its logs |
| `inparallel N` | Max simultaneous backup streams |
| `netusage N` | Total network bandwidth limit (KB/s) |
| `maxdumps N` | Max dumps per client simultaneously |
| `ssh_keys` | SSH private key for amandabackup connections |
| `compress client fast` | Compress on client with fast algorithm |
| `index yes` | Create per-file index (required for `amrecover`) |

[↑ Table of Contents](#table-of-contents)

---

## 7. disklist — Defining What to Back Up

The `disklist` file defines the **Disk List Entries (DLEs)** — what directories to back up from which clients.

```
FORMAT: hostname  diskname  dumptype
```

```bash
sudo tee /etc/amanda/DailySet/disklist <<'EOF'
# /etc/amanda/DailySet/disklist
# Format: client    diskname    dumptype
#
# Each line defines one DLE (Disk List Entry)
# diskname is either an actual path or a logical name

# ─── Local server (backup-server itself) ─────────────────────────
backup-server    /etc          comp-root
backup-server    /home         comp-user
backup-server    /root         comp-root
backup-server    /boot         comp-root
backup-server    /var/www      comp-user
backup-server    /opt          comp-user
backup-server    /srv          comp-user
backup-server    /usr/local    comp-root

# ─── Remote client 1 ─────────────────────────────────────────────
backup-client-1  /etc          comp-root
backup-client-1  /home         comp-user
backup-client-1  /root         comp-root
backup-client-1  /var/www      comp-user
EOF

sudo chown amandabackup:disk /etc/amanda/DailySet/disklist
```

### disklist with per-DLE options

You can override dumptype options per-DLE:

```
# Back up /var/lib but skip indexing (large, hard to browse)
backup-server    /var/lib    {
    no-index
    compress client best
    priority medium
}

# Back up a network filesystem (no compression — latency sensitive)
backup-server    /nfs/share    {
    global
    compress none
}
```

[↑ Table of Contents](#table-of-contents)

---

## 8. Initialise Amanda

```bash
# Label virtual tapes
# Amanda needs labelled tapes before it can use them
sudo -u amandabackup amadmin DailySet tape

# Or label them manually
for i in $(seq -w 01 10); do
  sudo -u amandabackup amlabel DailySet "DailySet-${i}" slot ${i}
done
```

[↑ Table of Contents](#table-of-contents)

---

## 9. amcheck — Pre-backup Validation

`amcheck` verifies that everything is configured correctly before running a backup. Always run `amcheck` before the first backup and after config changes.

```bash
# Run amcheck as amandabackup user
sudo -u amandabackup amcheck DailySet

# What amcheck tests:
# - Can Amanda connect to each client via SSH?
# - Does amandad exist on each client?
# - Are the configured disk paths accessible?
# - Is there enough free space in the VTL?
# - Are tape labels correct?

# Amcheck with verbose output
sudo -u amandabackup amcheck -v DailySet

# Check a specific client only
sudo -u amandabackup amcheck -h backup-client-1 DailySet
```

**Expected output when healthy:**
```
Amanda Tape Server Host Check
-----------------------------
  Dump host backup-server: OK
  Tape changer: OK
  Tape DailySet-01 is ok.
...

Amanda Client Host Check
------------------------
  backup-server: /etc: OK
  backup-server: /home: OK
  backup-client-1: /etc: OK
...

(brought to you by Amanda version 3.5.x)
```

[↑ Table of Contents](#table-of-contents)

---

## 10. amdump — Running Backups

```bash
# Run the Amanda backup
sudo -u amandabackup amdump DailySet

# Amanda automatically decides which DLEs to dump as Full vs Incremental
# based on the dumpcycle, runspercycle, and last dump history

# Run with verbose output to console
sudo -u amandabackup amdump DailySet --no-taper

# Dry run — estimate what would be backed up
sudo -u amandabackup amadmin DailySet estimate
```

### What amdump does

1. Reads `amanda.conf` and `disklist`
2. Consults `curinfo` for each DLE's history
3. Determines optimal level (Full or Incremental) for each DLE
4. Runs dumps in parallel (up to `inparallel`)
5. Streams data to VTL slots
6. Updates `curinfo` with statistics
7. Builds file indexes (if `index yes`)
8. Sends report email

[↑ Table of Contents](#table-of-contents)

---

## 11. Amanda Report

After each amdump run, Amanda generates a detailed report:

```bash
# View the latest Amanda report
sudo -u amandabackup amreport DailySet

# View a specific report by date
sudo -u amandabackup amreport DailySet --log=/var/log/amanda/DailySet/log.20260224

# List all available logs
ls /var/log/amanda/DailySet/
```

**Sample report output:**
```
Hostname        Disk              Level  Orig KB  Out KB  Comp%  Tape
--------------- ----------------- -----  -------- ------- ------ ---------
backup-server   /etc              1       8192    3120    38.1%  DailySet-01
backup-server   /home             0      51200   18432    36.0%  DailySet-01
backup-client-1 /etc              1       6144    2304    37.5%  DailySet-01
```

[↑ Table of Contents](#table-of-contents)

---

## 12. amadmin — Administration Tool

```bash
# Show Amanda's current state and tape status
sudo -u amandabackup amadmin DailySet info

# Show which tapes have been used
sudo -u amandabackup amadmin DailySet tape

# Show what would happen on next dump (without running it)
sudo -u amandabackup amadmin DailySet balance

# Show statistics for a specific DLE
sudo -u amandabackup amadmin DailySet info backup-server /etc

# Force a full dump on next run for a specific DLE
sudo -u amandabackup amadmin DailySet force backup-server /etc

# Force incremental on next run
sudo -u amandabackup amadmin DailySet force-level 1 backup-server /home

# Reset statistics for a DLE (forces full dump calculation)
sudo -u amandabackup amadmin DailySet delete backup-server /etc
# Then re-add it to curinfo by running amdump

# Reuse a tape now (don't wait for tapecycle)
sudo -u amandabackup amadmin DailySet reuse DailySet-05
```

[↑ Table of Contents](#table-of-contents)

---

## 13. amrestore — Restoring Files

### 13.1 Find which tape a file is on

```bash
# Search the index for a specific file
sudo -u amandabackup amadmin DailySet find /etc/ssh/sshd_config
# Output shows: date, client, diskname, level, tape label, file position
```

### 13.2 amrestore — restore from tape/VTL

```bash
# Restore to current directory
# (First: load the right VTL slot)
sudo -u amandabackup amlabel DailySet -l DailySet-01

# Basic restore of a full dump
cd /restore/amanda
sudo -u amandabackup amrestore \
  -p DailySet DailySet-01 | tar xf -

# Restore specific files/paths
sudo -u amandabackup amrestore \
  -p DailySet DailySet-01 | tar xf - ./etc/nginx/
```

### 13.3 amrecover — Interactive recovery (recommended)

`amrecover` is the interactive restore tool for Amanda. It is the easiest way to recover specific files.

```bash
# On the backup server, connect to amrecover
sudo -u amandabackup amrecover DailySet

# amrecover interactive commands:
# sethost backup-client-1    - switch to a client
# setdisk /etc               - switch to a DLE
# setdate 2026-02-17         - pick a date
# ls                         - list files
# cd nginx                   - change directory
# add nginx.conf             - mark file for restore
# add *                      - mark all files in current directory
# list                       - show marked files
# extract                    - restore marked files
# quit                       - exit
```

**Full amrecover session:**
```bash
sudo -u amandabackup amrecover DailySet
# amrecover> sethost backup-server
# amrecover> setdisk /etc
# amrecover> setdate 2026-02-17
# amrecover> ls
# amrecover> cd ssh
# amrecover> ls
#   sshd_config
#   ssh_host_ed25519_key
#   ...
# amrecover> add sshd_config
# amrecover> list
# amrecover> extract
#   Restoring files using tape drive tpchanger from host backup-server
#   Load tape DailySet-01 now
#   Continue [Y/n]? Y
# amrecover> quit
```

[↑ Table of Contents](#table-of-contents)

---

## 14. Automating Amanda with Cron

Amanda is typically run via cron as the `amandabackup` user:

```bash
# Add to amandabackup's crontab
sudo -u amandabackup crontab -e

# Example crontab entries:
# Run amdump every night at 01:00
# Run amcheck every night at 00:00 (1 hour before backup)
```

```cron
# Amanda backup cron jobs
# Check at midnight
0 0 * * * /usr/sbin/amcheck -m DailySet

# Run backup at 1am
0 1 * * * /usr/sbin/amdump DailySet

# Send weekly report on Sunday
0 8 * * 0 /usr/sbin/amreport DailySet | mail -s "Amanda Weekly Report" root@localhost

# Monthly cleanup of old logs
0 3 1 * * find /var/log/amanda/DailySet -name "log.*" -mtime +90 -delete
```

[↑ Table of Contents](#table-of-contents)

---

## 15. Multiple Configuration Sets

Amanda supports multiple independent config sets. Create one per backup schedule or purpose:

```bash
# DailySet — daily incrementals, 7-day cycle
/etc/amanda/DailySet/

# WeeklySet — weekly fulls, longer retention
/etc/amanda/WeeklySet/

# DatabaseSet — hourly database dumps
/etc/amanda/DatabaseSet/
```

Each set has its own `amanda.conf`, `disklist`, VTL, and cron entry.

[↑ Table of Contents](#table-of-contents)

---

## Lab Exercises

### Lab 08-1: Install and configure Amanda server

```bash
# 1. Install Amanda
sudo dnf install -y amanda amanda-server amanda-client

# 2. Create VTL directory structure
sudo mkdir -p /backup/vtl/DailySet
sudo chown amandabackup:disk /backup/vtl/DailySet
sudo -u amandabackup bash -c 'for i in $(seq -w 01 05); do mkdir -p /backup/vtl/DailySet/slot${i}; done'

# 3. Create config directory
sudo mkdir -p /etc/amanda/DailySet
sudo chown amandabackup:disk /etc/amanda/DailySet

# 4. Create log and state dirs
sudo mkdir -p /var/log/amanda/DailySet /var/lib/amanda/DailySet/{curinfo,index}
sudo chown -R amandabackup:disk /var/log/amanda /var/lib/amanda/DailySet
```

### Lab 08-2: Write amanda.conf and disklist

```bash
# 1. Create amanda.conf (use Section 6 as template)
# 2. Create disklist with at least 2 DLEs from the local server
# 3. Label the VTL tapes
sudo -u amandabackup amlabel DailySet DailySet-01 slot 01
sudo -u amandabackup amlabel DailySet DailySet-02 slot 02

# 4. Run amcheck
sudo -u amandabackup amcheck DailySet
```

### Lab 08-3: Run first backup

```bash
# 1. Run amdump
sudo -u amandabackup amdump DailySet

# 2. Watch the log
sudo tail -f /var/log/amanda/DailySet/log.$(date +%Y%m%d*)

# 3. View the report
sudo -u amandabackup amreport DailySet
```

### Lab 08-4: Interactive restore with amrecover

```bash
# 1. Create a test file
echo "Amanda restore test $(date)" | sudo tee /etc/amanda-test.txt

# 2. Run a backup
sudo -u amandabackup amdump DailySet

# 3. Delete the file
sudo rm /etc/amanda-test.txt

# 4. Recover it with amrecover
sudo -u amandabackup amrecover DailySet
# > sethost backup-server
# > setdisk /etc
# > add amanda-test.txt
# > extract
# > quit

# 5. Verify
cat /etc/amanda-test.txt
```

### Lab 08-5: Set up cron automation

```bash
sudo -u amandabackup crontab -l 2>/dev/null || true

sudo -u amandabackup crontab - <<'EOF'
0 0 * * * /usr/sbin/amcheck -m DailySet
0 1 * * * /usr/sbin/amdump DailySet
EOF

sudo -u amandabackup crontab -l
```

[↑ Table of Contents](#table-of-contents)

---

## Review Questions

1. What is a DLE in Amanda terminology?
2. How does Amanda decide whether to run a Full or Incremental backup for a DLE?
3. What does `amcheck` verify, and when should you run it?
4. What is the difference between `amrestore` and `amrecover`?
5. What does the `tapecycle` directive control?
6. What does `index yes` do in a dumptype, and which tool requires it?
7. How does Amanda authenticate to clients — what mechanism does it use?
8. What is a Virtual Tape Library (VTL) in Amanda?
9. What does `amadmin DailySet force backup-server /etc` do?
10. How do you view the report from last night's Amanda backup?

[↑ Table of Contents](#table-of-contents)

---

## Answers to Review Questions

1. A **DLE (Disk List Entry)** is a single line in the `disklist` file defining one directory on one client to be backed up. Each DLE is backed up independently with its own history and dump level tracking.
2. Amanda tracks the history of each DLE in `curinfo`. Based on `dumpcycle` (how often a full is desired) and `runspercycle` (how many runs per cycle), Amanda calculates which DLEs are overdue for a full backup and promotes them. Others get incrementals. This **automatic balancing** distributes full backups across the week.
3. `amcheck` verifies: SSH connectivity to each client, existence of `amandad` on clients, accessibility of all DLE paths, adequate space in the VTL/tape device, and correct tape labels. Run it before the first backup, after config changes, and ideally before every scheduled backup run (as a pre-check cron job).
4. `amrestore` reads raw Amanda archive data from a tape/VTL and pipes it as a tar stream — you pipe it to `tar xf -` to extract. `amrecover` is an interactive tool that queries the Amanda catalog index, lets you browse the backup like a filesystem, mark files, and extract them — without manually knowing which tape/position to use.
5. `tapecycle N` defines how many tape slots Amanda rotates through before reusing the oldest one. For example, `tapecycle 10` means Amanda will cycle through 10 tapes before overwriting the first one, giving approximately 10 backup generations.
6. `index yes` instructs Amanda to create a per-file index of every file in the backup. This index is stored in `indexdir` and is required by `amrecover` to browse and restore individual files. Without it, you can only restore entire DLEs.
7. Amanda authenticates to clients via **SSH**, using key-based authentication (the `amandabackup` user's SSH private key). The client's `authorized_keys` file has the server's public key, optionally restricted with `command=` to only allow `amandad`.
8. A **VTL (Virtual Tape Library)** is a directory on disk that Amanda treats as a tape library. Each subdirectory (`slot01`, `slot02`...) is a virtual tape slot. This allows Amanda to be used with disk storage while keeping its tape-based design and rotation logic.
9. `amadmin DailySet force backup-server /etc` marks that DLE as requiring a **Full backup on the next amdump run**, overriding Amanda's normal scheduling calculation. Useful after a system change, restore, or when you know the incremental chain is broken.
10. `sudo -u amandabackup amreport DailySet` — shows the report for the most recent backup run. Older reports can be viewed with `--log=/var/log/amanda/DailySet/log.YYYYMMDD`.

[↑ Table of Contents](#table-of-contents)

---

*Previous: [07 — Bareos](07-bareos.md)*
*Next: [09 — Strategy, Automation & Retention](09-strategy-automation.md)*

© 2026 UncleJS — Licensed under CC BY-NC-SA 4.0
