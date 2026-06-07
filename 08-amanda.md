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
  - [3.6 Configure amandad invocation (on clients)](#36-configure-amandad-invocation-on-clients)
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
  - [13.1 Find which dumps exist for a DLE](#131-find-which-dumps-exist-for-a-dle)
  - [13.2 amfetchdump — restore a dump image from the VTL](#132-amfetchdump--restore-a-dump-image-from-the-vtl)
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
| **On-demand agent** | Clients run `amandad`, invoked per-run (via SSH in this module) rather than as a persistent daemon |

### Amanda vs Bareos

| | Amanda | Bareos |
|--|--------|--------|
| Architecture | Server-centric, SSH-based | Daemons on server and all clients |
| Catalog | Simple text files + index | SQL database (PostgreSQL) |
| Configuration | Two files (amanda.conf + disklist) | Directory of resource files |
| GUI | Zmanda Management Console (commercial) | bareos-webui (included) |
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

> **⚠️ RHEL 10 availability:** as of mid-2026, Amanda is packaged in **EPEL 9 but not EPEL 10** — `dnf install amanda-server` fails on a stock RHEL 10 system. Options: run the Amanda server on a RHEL 9 host (EPEL 9 has `amanda-3.5.3`), rebuild the EPEL 9 source RPM for EL10, or build from source (github.com/zmanda/amanda). The configuration in this module is identical regardless of how Amanda was installed. Check `dnf info amanda-server` — if it appears in EPEL 10 later, install normally.

```bash
# Enable EPEL
sudo dnf install -y epel-release

# Install Amanda server and client packages (see availability note above)
sudo dnf install -y amanda-server amanda-client

# Verify the installed version
rpm -q amanda-server
sudo -u amandabackup amgetconf build.version 2>/dev/null || rpm -q --qf '%{VERSION}\n' amanda-libs
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

### 3.6 Configure amandad invocation (on clients)

RHEL 10 ships **no xinetd package**, so the traditional inetd-style activation is unavailable. Launch `amandad` over SSH (recommended — used throughout this module) or write systemd socket units:

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

# Create virtual tape slots (10 slots; chg-disk expects UNPADDED names: slot1..slot10)
sudo -u amandabackup bash <<'EOF'
cd /backup/vtl/DailySet
for i in $(seq 1 10); do
  mkdir -p slot${i}
done
# chg-disk uses a 'data' symlink pointing at the currently loaded slot
ln -sf slot1 data
EOF

ls -la /backup/vtl/DailySet/
```

### Configure the vtape (virtual tape) changer

No separate changer file is needed — the modern `chg-disk:` changer is configured
directly in `amanda.conf` (Section 6):

```
tpchanger "chg-disk:/backup/vtl/DailySet"
property "num-slot" "10"
property "auto-create-slot" "yes"
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

# Where Amanda mails its reports
mailto "root@localhost"

# Scheduling cycle
dumpcycle 7            # Aim for a full dump of every DLE within 7 days
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
property "num-slot" "10"                       # Number of VTL slots
property "auto-create-slot" "yes"              # Create slot dirs on demand
tapetype "HARDDISK"
labelstr "^DailySet-[0-9][0-9]*$"              # Regex: valid tape labels

# Tape (VTL slot) definition
define tapetype HARDDISK {
  comment "Disk-based virtual tape"
  length  10 gbytes    # Size of each VTL slot
  filemark  0 kbytes   # No physical filemark for disk
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

# Reserve space for Amanda's own operations
reserved-space 100 mbytes

# Dumptypes — define backup profiles
# (auth/ssh_keys/amandad_path/maxdumps are per-dumptype settings — see 'global' below)
define dumptype global {
  comment "Global defaults"
  compress client fast
  auth "ssh"
  ssh_keys "/var/lib/amanda/.ssh/id_ed25519"
  amandad_path "/usr/libexec/amanda/amandad"
  index yes                  # Create file index (enables amrecover)
  estimate server            # Estimate method
  maxdumps 2                 # Max concurrent dumps from one client (a count, not a time)
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
backup-client  /etc          comp-root
backup-client  /home         comp-user
backup-client  /root         comp-root
backup-client  /var/www      comp-user
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
# Label the virtual tapes — Amanda needs labelled tapes before it can use them
# (slot numbers are unpadded: slot 1..slot 10; pad only the LABEL for sorting)
for i in $(seq 1 10); do
  sudo -u amandabackup amlabel DailySet "DailySet-$(printf '%02d' $i)" slot ${i}
done

# Verify: show which tape Amanda expects to write next
sudo -u amandabackup amadmin DailySet tape
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
sudo -u amandabackup amcheck -h backup-client DailySet
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
  backup-client: /etc: OK
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

# Dump to the holding disk only — skip writing to tape (flush later with amflush)
sudo -u amandabackup amdump DailySet --no-taper

# Show the server's stored size estimates per DLE
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
backup-client /etc              1       6144    2304    37.5%  DailySet-01
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

# Force a level-1 incremental on next run (note: one token, hyphenated)
sudo -u amandabackup amadmin DailySet force-level-1 backup-server /home

# ⚠️ Remove a DLE from Amanda's database entirely (destroys its history —
# only use when permanently retiring a DLE; also remove it from disklist)
sudo -u amandabackup amadmin DailySet delete backup-server /etc

# Reuse a tape now (don't wait for tapecycle)
sudo -u amandabackup amadmin DailySet reuse DailySet-05
```

[↑ Table of Contents](#table-of-contents)

---

## 13. amrestore — Restoring Files

### 13.1 Find which dumps exist for a DLE

```bash
# amadmin find matches host/disk (DLEs), not individual file paths
sudo -u amandabackup amadmin DailySet find backup-server /etc
# Output shows: date, client, diskname, level, tape label, file position
# (To locate a single FILE, use amrecover's indexed browsing — 13.3)
```

### 13.2 amfetchdump — restore a dump image from the VTL

```bash
# amfetchdump locates the right vtape via the changer and writes the
# dump image — no manual slot loading needed
cd /restore/amanda
sudo -u amandabackup amfetchdump -p DailySet backup-server /etc | tar xf -

# Restore specific files/paths from the image
sudo -u amandabackup amfetchdump -p DailySet backup-server /etc | tar xf - ./etc/nginx/
```

### 13.3 amrecover — Interactive recovery (recommended)

`amrecover` is the interactive restore tool for Amanda. It is the easiest way to recover specific files.

> **Setup required:** `amrecover` talks to the index server (`amindexd`) and tape server (`amidxtaped`) services. On RHEL 10 (no xinetd) these need systemd socket units, or run `amrecover` directly on the backup server as root with `index_server`/`tape_server` set to `localhost` in `/etc/amanda/amanda-client.conf`.

```bash
# On the backup server, connect to amrecover
sudo -u amandabackup amrecover DailySet

# amrecover interactive commands:
# sethost backup-client    - switch to a client
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
sudo dnf install -y amanda-server amanda-client   # see §3.1 RHEL 10 availability note

# 2. Create VTL directory structure (chg-disk expects unpadded slot names)
sudo mkdir -p /backup/vtl/DailySet
sudo chown amandabackup:disk /backup/vtl/DailySet
sudo -u amandabackup bash -c 'for i in $(seq 1 5); do mkdir -p /backup/vtl/DailySet/slot${i}; done'

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
# 3. Label the VTL tapes (slot numbers are unpadded)
sudo -u amandabackup amlabel DailySet DailySet-01 slot 1
sudo -u amandabackup amlabel DailySet DailySet-02 slot 2

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
8. A **VTL (Virtual Tape Library)** is a directory on disk that Amanda treats as a tape library. Each subdirectory (`slot1`, `slot2`...) is a virtual tape slot. This allows Amanda to be used with disk storage while keeping its tape-based design and rotation logic.
9. `amadmin DailySet force backup-server /etc` marks that DLE as requiring a **Full backup on the next amdump run**, overriding Amanda's normal scheduling calculation. Useful after a system change, restore, or when you know the incremental chain is broken.
10. `sudo -u amandabackup amreport DailySet` — shows the report for the most recent backup run. Older reports can be viewed with `--log=/var/log/amanda/DailySet/log.YYYYMMDD`.

[↑ Table of Contents](#table-of-contents)

---

*Previous: [07 — Bareos](07-bareos.md)*
*Next: [09 — Strategy, Automation & Retention](09-strategy-automation.md)*

© 2026 UncleJS — Licensed under CC BY-NC-SA 4.0
