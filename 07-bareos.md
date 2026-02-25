# Module 07 — Bareos — Enterprise Backup (Full Install & Deep Configuration)

## Learning Objectives

By the end of this module you will be able to:

- Explain the Bareos architecture: Director, Storage Daemon, File Daemon, Catalog
- Install and configure a complete Bareos server stack on RHEL 10
- Configure MariaDB as the Bareos catalog database
- Write and understand every resource in the Director, Storage, and File Daemon configs
- Create Jobs, JobDefs, Schedules, FileSets, Pools, and Storage resources
- Add and manage multiple clients
- Use bconsole for all administrative operations
- Configure TLS encryption between all Bareos daemons
- Restore files using the interactive restore wizard and bextract
- Maintain the catalog database
- Monitor and alert on backup job status
- Install and configure the Bareos WebUI with Apache and PHP-FPM
- Create WebUI console users with full, limited, and read-only access profiles
- Perform backups and restores through the WebUI file-tree browser

---

## Table of Contents

- [1. Bareos Architecture](#1-bareos-architecture)
  - [Component descriptions](#component-descriptions)
  - [Communication flow for a backup job](#communication-flow-for-a-backup-job)
  - [WebUI communication flow](#webui-communication-flow)
- [2. Installation](#2-installation)
  - [2.1 Add the Bareos repository](#21-add-the-bareos-repository)
  - [2.2 Install Bareos packages](#22-install-bareos-packages)
  - [2.3 Configure and start MariaDB](#23-configure-and-start-mariadb)
  - [2.4 Create the Bareos catalog database](#24-create-the-bareos-catalog-database)
  - [2.5 Configure firewall](#25-configure-firewall)
  - [2.6 Configure SELinux](#26-configure-selinux)
  - [2.7 Enable and start all Bareos services](#27-enable-and-start-all-bareos-services)
- [3. Configuration Directory Structure](#3-configuration-directory-structure)
- [4. Director Configuration — Deep Dive](#4-director-configuration--deep-dive)
  - [4.1 Director self-reference (`bareos-dir.d/director/bareos-dir.conf`)](#41-director-self-reference-bareos-dirdirectorbareos-dirconf)
  - [4.2 Catalog resource (`bareos-dir.d/catalog/MyCatalog.conf`)](#42-catalog-resource-bareos-dircatalogmycatalogconf)
  - [4.3 Messages resource (`bareos-dir.d/messages/Standard.conf`)](#43-messages-resource-bareos-dirmessagesstandardconf)
  - [4.4 JobDefs resource — reusable job templates](#44-jobdefs-resource--reusable-job-templates)
  - [4.5 Schedule resource (`bareos-dir.d/schedule/WeeklyCycle.conf`)](#45-schedule-resource-bareos-dirscheduleweeklycycleconf)
  - [4.6 FileSet resource (`bareos-dir.d/fileset/`)](#46-fileset-resource-bareos-dirfileset)
  - [4.7 Pool resource (`bareos-dir.d/pool/`)](#47-pool-resource-bareos-dirpool)
  - [4.8 Storage resource (`bareos-dir.d/storage/File.conf`)](#48-storage-resource-bareos-dirstoragefileconf)
  - [4.9 Client resource (`bareos-dir.d/client/bareos-fd.conf`)](#49-client-resource-bareos-dirclientbareos-fdconf)
  - [4.10 Job resource (`bareos-dir.d/job/`)](#410-job-resource-bareos-dirjob)
- [5. Storage Daemon Configuration](#5-storage-daemon-configuration)
  - [5.1 Storage Daemon self-definition (`bareos-sd.d/storage/bareos-sd.conf`)](#51-storage-daemon-self-definition-bareos-sdstoragebareos-sdconf)
  - [5.2 Authorise the Director to connect (`bareos-sd.d/director/bareos-dir.conf`)](#52-authorise-the-director-to-connect-bareos-sddirectorbareos-dirconf)
  - [5.3 Device resource — disk-based storage (`bareos-sd.d/device/FileStorage.conf`)](#53-device-resource--disk-based-storage-bareos-sddevicefilestorageconf)
- [6. File Daemon Configuration](#6-file-daemon-configuration)
  - [6.1 File Daemon self-definition (`bareos-fd.d/client/myself.conf`)](#61-file-daemon-self-definition-bareos-fdclientmyselfconf)
  - [6.2 Authorise the Director to connect (`bareos-fd.d/director/bareos-dir.conf`)](#62-authorise-the-director-to-connect-bareos-fddirectorbareos-dirconf)
- [7. Setting Passwords](#7-setting-passwords)
- [8. bconsole Configuration](#8-bconsole-configuration)
- [9. Validate and Reload Configuration](#9-validate-and-reload-configuration)
- [10. bconsole — Complete Command Reference](#10-bconsole--complete-command-reference)
  - [10.1 Connecting](#101-connecting)
  - [10.2 Status commands](#102-status-commands)
  - [10.3 Running jobs manually](#103-running-jobs-manually)
  - [10.4 Monitoring jobs](#104-monitoring-jobs)
  - [10.5 Listing volumes and files](#105-listing-volumes-and-files)
  - [10.6 Volume management](#106-volume-management)
  - [10.7 Pruning the catalog](#107-pruning-the-catalog)
  - [10.8 Cancelling jobs](#108-cancelling-jobs)
  - [10.9 Show config resources](#109-show-config-resources)
- [11. The Interactive Restore Wizard](#11-the-interactive-restore-wizard)
  - [Restore the latest backup of a client](#restore-the-latest-backup-of-a-client)
  - [Restore to a specific date](#restore-to-a-specific-date)
  - [Restore all files from the latest backup](#restore-all-files-from-the-latest-backup)
- [12. Adding a Remote Client](#12-adding-a-remote-client)
  - [12.1 Install File Daemon on the client](#121-install-file-daemon-on-the-client)
  - [12.2 Configure File Daemon on the client](#122-configure-file-daemon-on-the-client)
  - [12.3 Add the client to the Director](#123-add-the-client-to-the-director)
- [13. Database Maintenance](#13-database-maintenance)
  - [13.1 Automated catalog backup](#131-automated-catalog-backup)
  - [13.2 Manual catalog backup](#132-manual-catalog-backup)
  - [13.3 Catalog database check and repair](#133-catalog-database-check-and-repair)
  - [13.4 Catalog size management](#134-catalog-size-management)
  - [13.5 Pruning the catalog from bconsole](#135-pruning-the-catalog-from-bconsole)
- [14. TLS Configuration](#14-tls-configuration)
  - [14.1 Generate TLS certificates](#141-generate-tls-certificates)
  - [14.2 Enable TLS in Director](#142-enable-tls-in-director)
  - [14.3 Enable TLS in Storage Daemon](#143-enable-tls-in-storage-daemon)
  - [14.4 Enable TLS in File Daemon](#144-enable-tls-in-file-daemon)
  - [14.5 Verify TLS is active](#145-verify-tls-is-active)
- [15. Monitoring and Alerting](#15-monitoring-and-alerting)
  - [15.1 Email notifications](#151-email-notifications)
  - [15.2 systemd journal monitoring](#152-systemd-journal-monitoring)
  - [15.3 Bareos log file](#153-bareos-log-file)
  - [15.4 Monitor from bconsole](#154-monitor-from-bconsole)
- [16. bextract — Restore Without a Running Director](#16-bextract--restore-without-a-running-director)
- [17. Bareos WebUI](#17-bareos-webui)
  - [17.1 Architecture](#171-architecture)
  - [17.2 Features](#172-features)
  - [17.3 Install bareos-webui](#173-install-bareos-webui)
  - [17.4 SELinux — Required Boolean](#174-selinux--required-boolean)
  - [17.5 Apache Configuration](#175-apache-configuration)
  - [17.6 Director: Console Resource for WebUI](#176-director-console-resource-for-webui)
  - [17.7 Director: Profile Resources](#177-director-profile-resources)
  - [17.8 `/etc/bareos-webui/directors.ini` — Full Annotated Config](#178-etcbareos-webuilirectorsini--full-annotated-config)
  - [17.9 `/etc/bareos-webui/configuration.ini` — Full Annotated Config](#179-etcbareos-webuiconfigurationini--full-annotated-config)
  - [17.10 Bvfs Cache — Keep the File-Tree Browser Responsive](#1710-bvfs-cache--keep-the-file-tree-browser-responsive)
  - [17.11 TLS Between WebUI and Director (Certificate-Based)](#1711-tls-between-webui-and-director-certificate-based)
  - [17.12 NGINX Alternative](#1712-nginx-alternative)
  - [17.13 Multi-Director Setup](#1713-multi-director-setup)
  - [17.14 Accessing the WebUI](#1714-accessing-the-webui)
- [Lab Exercises](#lab-exercises)
  - [Lab 07-1: Complete Bareos installation](#lab-07-1-complete-bareos-installation)
  - [Lab 07-2: Configure a FileSet and Job](#lab-07-2-configure-a-fileset-and-job)
  - [Lab 07-3: Run a backup and verify](#lab-07-3-run-a-backup-and-verify)
  - [Lab 07-4: Restore a single file](#lab-07-4-restore-a-single-file)
  - [Lab 07-5: Add a second client](#lab-07-5-add-a-second-client)
  - [Lab 07-6: Configure TLS](#lab-07-6-configure-tls)
  - [Lab 07-7: Test bextract (bare-metal restore simulation)](#lab-07-7-test-bextract-bare-metal-restore-simulation)
  - [Lab 07-8: Configure email alerting](#lab-07-8-configure-email-alerting)
  - [Lab 07-9: Install and use Bareos WebUI](#lab-07-9-install-and-use-bareos-webui)
- [Review Questions](#review-questions)
- [Answers to Review Questions](#answers-to-review-questions)

---

## 1. Bareos Architecture

Bareos (Backup Archiving REcovery Open Sourced) is a network backup solution forked from Bacula. It uses a client/server architecture with separate components that communicate over TCP.

```
                          ┌──────────────────────────────────────────────────────────────────┐
                          │                         BACKUP SERVER                            │
                          │                                                                  │
  ┌──────────┐  HTTPS/80  │  ┌─────────────┐  PHP-FPM  ┌─────────────────┐                 │
  │ Browser  │───────────►│  │  Apache     │──────────►│  bareos-webui   │                 │
  └──────────┘            │  │  httpd      │           │  (PHP/ZF2 app)  │                 │
                          │  └─────────────┘           └────────┬────────┘                 │
                          │                                     │ TCP 9101                  │
                          │  ┌──────────────┐                   │                           │
  bconsole ───TCP 9101───►│  │   Director   │◄──────────────────┘                           │
                          │  │ (bareos-dir) │──────────────────────────────────────────┐    │
                          │  │  Port 9101   │   ┌──────────────┐   ┌────────────────┐  │    │
                          │  └──────┬───────┘   │   Storage    │   │    Catalog     │  │    │
                          │         │           │   Daemon     │◄──│   (MariaDB)    │  │    │
                          │         │           │ (bareos-sd)  │   │   Port 3306    │  │    │
                          │         │           │  Port 9103   │   └────────────────┘  │    │
                          │         │           └──────────────┘                       │    │
                          └─────────┼─────────────────────────────────────────────────-┘    │
                                    │  TCP 9102                                              │
                                    │                                                        │
                              ┌─────▼──────────┐        ┌──────────────────┐                │
                              │  File Daemon   │        │  File Daemon     │                │
                              │ (bareos-fd)    │        │ (bareos-fd)      │                │
                              │ backup-server  │        │ backup-client-1  │                │
                              │  (local)       │        │  (remote)        │                │
                              └────────────────┘        └──────────────────┘                │
```

### Component descriptions

| Component | Process | Port | Role |
|-----------|---------|------|------|
| **Director** | `bareos-dir` | 9101 | Brain of Bareos. Orchestrates all backup and restore operations. Reads config, schedules jobs, coordinates FD and SD. |
| **Storage Daemon** | `bareos-sd` | 9103 | Manages physical storage — disk files, tape drives. Receives data from FD, writes to storage devices. |
| **File Daemon** | `bareos-fd` | 9102 | Runs on every client. Reads files from the client filesystem and sends them to the Storage Daemon. |
| **Catalog** | MariaDB/MySQL | 3306 | Relational database storing metadata: what was backed up, when, where. NOT the backup data itself. |
| **bconsole** | `bconsole` | — | CLI management console. Connects directly to the Director on port 9101. |
| **Bareos WebUI** | `httpd` + `php-fpm` | 80/443 | Browser-based dashboard. Communicates with the Director over the same TCP 9101 bconsole protocol using a named Console resource. Installed separately — see Section 17. |

### Communication flow for a backup job

```
1. Director reads schedule → triggers Job
2. Director contacts File Daemon on client (port 9102) → authenticates
3. Director contacts Storage Daemon (port 9103) → prepares storage
4. File Daemon reads filesystem data
5. File Daemon sends data stream directly to Storage Daemon
6. Storage Daemon writes data to disk/tape
7. Storage Daemon confirms to Director
8. Director writes job metadata to Catalog (MariaDB)
9. Director sends email notification (if configured)
```

### WebUI communication flow

```
1. Administrator opens browser → HTTPS to Apache (port 80/443)
2. Apache passes request to PHP-FPM → bareos-webui PHP application
3. bareos-webui opens TCP connection to Director on port 9101
4. bareos-webui authenticates using a named Console resource (username + password)
5. bareos-webui issues bconsole commands (status, list jobs, restore, etc.)
6. Director responds with job data / file catalogue
7. bareos-webui formats response as HTML → Apache → browser
```

> **Note:** The WebUI does **not** store backup data and does **not** require direct access to the Storage Daemon. It is purely a management interface to the Director. It can run on the same host as the Director or on a separate web server.

[↑ Table of Contents](#table-of-contents)

---

## 2. Installation

### 2.1 Add the Bareos repository

```bash
# Import the Bareos repository for RHEL 10
# Check https://download.bareos.org/current/ for latest URLs
sudo dnf install -y wget

# Add Bareos repository (current stable release)
wget -O /etc/yum.repos.d/bareos.repo \
  https://download.bareos.org/current/EL_9/bareos.repo
# Note: Use EL_9 for RHEL 10 (AlmaLinux/Rocky) if EL_10 not yet available
# Check https://download.bareos.org/ for the exact RHEL 10 URL

# Or create the repo file manually
sudo tee /etc/yum.repos.d/bareos.repo <<'EOF'
[bareos]
name=Bareos EL $releasever
baseurl=https://download.bareos.org/current/EL_$releasever/$basearch/
enabled=1
gpgcheck=1
gpgkey=https://download.bareos.org/current/EL_$releasever/repodata/repomd.xml.key
EOF

# Import GPG key
sudo rpm --import https://download.bareos.org/current/EL_9/repodata/repomd.xml.key
```

### 2.2 Install Bareos packages

```bash
# Install Director, Storage Daemon, File Daemon, MariaDB catalog, and bconsole
sudo dnf install -y \
  bareos \
  bareos-director \
  bareos-storage \
  bareos-filedaemon \
  bareos-database-mysql \
  bareos-database-tools \
  bareos-bconsole \
  bareos-tools

# Also install MariaDB if not already present
sudo dnf install -y mariadb-server
```

### 2.3 Configure and start MariaDB

```bash
# Enable and start MariaDB
sudo systemctl enable --now mariadb

# Secure the installation
sudo mysql_secure_installation
# Set root password: yes
# Remove anonymous users: yes
# Disallow root login remotely: yes
# Remove test database: yes
# Reload privilege tables: yes
```

### 2.4 Create the Bareos catalog database

```bash
# Run the Bareos database creation script
# This creates the bareos database and schema
sudo -u bareos /usr/lib/bareos/scripts/create_bareos_database
sudo -u bareos /usr/lib/bareos/scripts/make_bareos_tables
sudo -u bareos /usr/lib/bareos/scripts/grant_bareos_privileges

# Verify the database was created
sudo mysql -u root -e "SHOW DATABASES;" | grep bareos
sudo mysql -u root -e "USE bareos; SHOW TABLES;"
```

### 2.5 Configure firewall

```bash
# Open Bareos daemon ports
sudo firewall-cmd --permanent --add-port=9101/tcp   # Director
sudo firewall-cmd --permanent --add-port=9102/tcp   # File Daemon
sudo firewall-cmd --permanent --add-port=9103/tcp   # Storage Daemon

# Open HTTP/HTTPS for Bareos WebUI (Apache)
sudo firewall-cmd --permanent --add-service=http    # port 80
sudo firewall-cmd --permanent --add-service=https   # port 443

sudo firewall-cmd --reload

# Verify
sudo firewall-cmd --list-ports
sudo firewall-cmd --list-services
```

> **WebUI note:** If the WebUI runs on a separate host from the Director, that host only needs ports 80/443 open to browsers. It must be able to reach the Director on TCP 9101 — no extra firewall rule is needed on the Director host if both are on the same network.

### 2.6 Configure SELinux

```bash
# Allow Bareos daemons to communicate on their ports
sudo setsebool -P bareos_use_connection 1 2>/dev/null || true

# If specific SELinux denials occur, check the audit log
sudo ausearch -c 'bareos-dir' --raw | audit2allow -M bareos_local
sudo semodule -i bareos_local.pp

# Bareos data directory SELinux context
sudo semanage fcontext -a -t bareos_store_t "/backup/bareos(/.*)?"
sudo restorecon -Rv /backup/bareos
```

**WebUI — additional SELinux boolean (required):**

```bash
# Allow Apache (httpd) to open network connections to the Director on TCP 9101
sudo setsebool -P httpd_can_network_connect 1

# Verify both booleans are on
getsebool bareos_use_connection httpd_can_network_connect
```

> Without `httpd_can_network_connect`, Apache will receive a TCP connection refused from SELinux when bareos-webui tries to reach the Director. The symptom is a "Can't connect to Director" error in the browser with nothing obviously wrong in the Bareos logs.

### 2.7 Enable and start all Bareos services

```bash
# Core Bareos daemons
sudo systemctl enable --now bareos-dir
sudo systemctl enable --now bareos-sd
sudo systemctl enable --now bareos-fd

# Check status
sudo systemctl status bareos-dir bareos-sd bareos-fd

# View startup logs
sudo journalctl -u bareos-dir -n 30
sudo journalctl -u bareos-sd -n 30
sudo journalctl -u bareos-fd -n 30
```

**Bareos WebUI services** (if installing WebUI on this host — see Section 17 for full setup):

```bash
# PHP-FPM must start before Apache
sudo systemctl enable --now php-fpm
sudo systemctl enable --now httpd

# Verify all five services are active
sudo systemctl is-active bareos-dir bareos-sd bareos-fd php-fpm httpd
```

| Service | Purpose |
|---------|---------|
| `bareos-dir` | Director daemon — job scheduling and orchestration |
| `bareos-sd` | Storage Daemon — writes backup data to disk/tape |
| `bareos-fd` | File Daemon — reads client filesystems |
| `php-fpm` | PHP FastCGI Process Manager — runs the WebUI PHP application |
| `httpd` | Apache web server — serves the WebUI to browsers |

[↑ Table of Contents](#table-of-contents)

---

## 3. Configuration Directory Structure

Bareos uses a directory-based configuration (since version 16+):

```
/etc/bareos/
├── bareos-dir.conf           # Main Director config (includes subdirs)
├── bareos-sd.conf            # Storage Daemon config
├── bareos-fd.conf            # File Daemon config (on this machine)
├── bconsole.conf             # bconsole client config
│
├── bareos-dir.d/             # Director config fragments
│   ├── catalog/              # Catalog database connection
│   │   └── MyCatalog.conf
│   ├── client/               # Client (File Daemon) definitions
│   │   └── bareos-fd.conf    # Local FD (created at install)
│   ├── console/              # bconsole + WebUI access definitions
│   │   ├── bareos-mon.conf   # Default monitor console
│   │   └── admin.conf        # WebUI admin console (Section 17.6)
│   ├── director/             # Director self-reference
│   │   └── bareos-dir.conf
│   ├── fileset/              # FileSets (what to back up)
│   │   ├── Catalog.conf
│   │   └── SelfTest.conf
│   ├── job/                  # Job definitions
│   │   ├── BackupCatalog.conf
│   │   └── RestoreFiles.conf
│   ├── jobdefs/              # JobDefs (reusable job templates)
│   │   └── DefaultJob.conf
│   ├── messages/             # Messages (notifications)
│   │   ├── Daemon.conf
│   │   └── Standard.conf
│   ├── pool/                 # Pool definitions
│   │   ├── Full.conf
│   │   ├── Incremental.conf
│   │   └── Differential.conf
│   ├── profile/              # Director access profiles (used by WebUI)
│   │   ├── operator.conf     # Default operator profile
│   │   ├── webui-admin.conf  # Full WebUI access (Section 17.7)
│   │   └── webui-readonly.conf # Read-only WebUI profile
│   ├── schedule/             # Schedule definitions
│   │   └── WeeklyCycle.conf
│   └── storage/              # Storage resource definitions
│       └── File.conf
│
├── bareos-sd.d/              # Storage Daemon config fragments
│   ├── autochanger/
│   ├── device/
│   │   └── FileStorage.conf
│   ├── director/
│   │   └── bareos-dir.conf   # Authorises Director to connect
│   ├── messages/
│   │   └── Standard.conf
│   └── storage/
│       └── bareos-sd.conf    # SD self-definition
│
└── bareos-fd.d/              # File Daemon config fragments
    ├── client/
    │   └── myself.conf       # FD self-definition
    ├── director/
    │   └── bareos-dir.conf   # Authorises Director to connect
    └── messages/
        └── Standard.conf

/etc/bareos-webui/            # WebUI application config (separate package)
├── directors.ini             # Director connection list (Section 17.8)
│                             #   [director-name]
│                             #   enabled = yes
│                             #   diraddress = 127.0.0.1
│                             #   dirport = 9101
│                             #   ...
└── configuration.ini         # WebUI behaviour settings (Section 17.9)
                              #   session lifetime, restore limits, debug, etc.
```

[↑ Table of Contents](#table-of-contents)

---

## 4. Director Configuration — Deep Dive

### 4.1 Director self-reference (`bareos-dir.d/director/bareos-dir.conf`)

```bash
sudo tee /etc/bareos/bareos-dir.d/director/bareos-dir.conf <<'EOF'
Director {
  # The name of this Director — must match in FD and SD configs
  Name = bareos-dir

  # Port the Director listens on for bconsole and FD connections
  QueryFile = "/usr/lib/bareos/scripts/query.sql"
  Maximum Concurrent Jobs = 10

  # TLS configuration (covered in Section 14)
  # TLS Enable = yes
  # TLS Require = yes
  # TLS CA Certificate File = /etc/bareos/tls/ca.crt
  # TLS Certificate = /etc/bareos/tls/director.crt
  # TLS Key = /etc/bareos/tls/director.key

  # Audit log for all bconsole commands
  Audit Log File = /var/log/bareos/bareos-audit.log
}
EOF
```

### 4.2 Catalog resource (`bareos-dir.d/catalog/MyCatalog.conf`)

The Catalog resource defines the database connection:

```bash
sudo tee /etc/bareos/bareos-dir.d/catalog/MyCatalog.conf <<'EOF'
Catalog {
  Name = MyCatalog

  # Database type: mysql (covers MariaDB)
  DB Driver = mysql

  # Database name (created in step 2.4)
  DB Name = bareos

  # Database host — localhost if MariaDB is on the same machine
  DB Address = localhost

  # Database port
  DB Port = 3306

  # Database user and password
  # This user was created by grant_bareos_privileges
  DB User = bareos
  DB Password = ""    # Bareos uses Unix socket auth by default
}
EOF
```

### 4.3 Messages resource (`bareos-dir.d/messages/Standard.conf`)

Messages define where job status notifications go:

```bash
sudo tee /etc/bareos/bareos-dir.d/messages/Standard.conf <<'EOF'
Messages {
  Name = Standard

  # Send email on job errors and warnings
  # Replace with your real email address
  MailCommand = "/usr/sbin/bsmtp -h localhost -f \"\(Bareos\) \<bareos@%r\>\" -s \"Bareos: %t %e of %c %l\" %r"
  Mail On Error = root@localhost = all, !skipped, !restored

  # Log all messages to the catalog
  Append = "/var/log/bareos/bareos.log" = all, !skipped, !saved, !audit

  # Send to Director's own log
  Director = bareos-dir = all, !skipped, !restored

  # Console output (visible in bconsole)
  Console = all, !skipped, !saved
}
EOF
```

### 4.4 JobDefs resource — reusable job templates

`JobDefs` (Job Definitions) are templates that Jobs inherit from. This avoids repetition.

```bash
sudo tee /etc/bareos/bareos-dir.d/jobdefs/DefaultJob.conf <<'EOF'
JobDefs {
  Name = "DefaultJob"

  # Type: Backup, Restore, Verify, Admin
  Type = Backup

  # Backup level default (overridden by Schedule)
  Level = Incremental

  # Default FileSet for this job template
  FileSet = "DefaultFileSet"

  # Default Schedule
  Schedule = "WeeklyCycle"

  # Which Storage resource to use
  Storage = FileStorage

  # Which Messages resource to use
  Messages = Standard

  # Which Pool to use for Full backups
  Pool = Full

  # Which Pool to use for Incremental backups
  Full Backup Pool = Full
  Incremental Backup Pool = Incremental
  Differential Backup Pool = Differential

  # Which Catalog to use
  Catalog = MyCatalog

  # How long to keep job records in the catalog
  # (not the actual backup data — that's controlled by Pool)
  Job Retention = 6 months

  # How long to keep File records in catalog (more disk space)
  # Can be shorter than Job Retention
  File Retention = 60 days

  # Auto-prune old jobs and files from catalog
  Auto Prune = yes

  # Maximum run time for the job
  Maximum Run Time = 24 hours

  # Run backup from client's Write Ahead Log for consistency
  # Useful with script to flush databases before backup
  # Client Run Before Job = "/usr/local/bin/pre-backup.sh"
  # Client Run After Job  = "/usr/local/bin/post-backup.sh"

  # Priority: lower number = higher priority
  Priority = 10
}
EOF
```

### 4.5 Schedule resource (`bareos-dir.d/schedule/WeeklyCycle.conf`)

```bash
sudo tee /etc/bareos/bareos-dir.d/schedule/WeeklyCycle.conf <<'EOF'
Schedule {
  Name = "WeeklyCycle"

  # Full backup: first Sunday of each month at 01:05
  Run = Full 1st sun at 01:05

  # Differential backup: every Sunday (except 1st) at 01:05
  Run = Differential 2nd-5th sun at 01:05

  # Incremental backup: Monday–Saturday at 01:05
  Run = Incremental mon-sat at 01:05
}

# Daily-only schedule (simpler)
Schedule {
  Name = "DailyFull"
  Run = Full daily at 02:00
}

# Hourly schedule for critical data
Schedule {
  Name = "HourlyCycle"
  Run = Incremental hourly at 0:05
}
EOF
```

**Schedule run syntax:**
```
Run = LEVEL  [day-specification] at HH:MM
Run = Level  [MODIFIER] weekday|monthday at time

Modifiers:
  1st, 2nd, 3rd, 4th, 5th
  first, second, third, fourth, fifth, last

Weekdays: mon, tue, wed, thu, fri, sat, sun
Ranges:   mon-fri, mon-sat

Examples:
  Run = Full 1st sun at 01:00
  Run = Incremental mon-sat at 02:00
  Run = Differential 2nd-5th sun at 01:00
  Run = Full last day at 23:00
  Run = Full daily at 03:00
  Run = Incremental hourly at 0:15
```

### 4.6 FileSet resource (`bareos-dir.d/fileset/`)

The FileSet defines exactly which files to back up and which to exclude.

```bash
sudo tee /etc/bareos/bareos-dir.d/fileset/DefaultFileSet.conf <<'EOF'
FileSet {
  Name = "DefaultFileSet"

  Include {
    # Options apply to all included paths
    Options {
      # Signature for file integrity verification
      Signature = MD5

      # Compression: GZIP, GZIP1-GZIP9, LZO, LZFAST, LZ4, LZ4HC
      Compression = GZIP6

      # Do not follow symlinks to other filesystems
      One FS = yes

      # Preserve access time (atime) — set to no to avoid spurious changes
      No Atime = yes

      # Handle sparse files efficiently
      Sparse = yes

      # Back up extended attributes (includes SELinux labels)
      XAttr = yes

      # Back up ACLs
      ACL Support = yes
    }

    # Paths to back up
    File = /etc
    File = /root
    File = /home
    File = /boot
    File = /var/www
    File = /var/lib/mysql     # Only if using consistent backup
    File = /opt
    File = /srv
    File = /usr/local
  }

  Exclude {
    # Directories to skip entirely
    File = /proc
    File = /sys
    File = /dev
    File = /run
    File = /tmp
    File = /var/tmp
    File = /var/cache
    File = /media
    File = /mnt
    File = /backup
    File = /var/lib/containers/storage/overlay
  }
}
EOF
```

```bash
# FileSet with Wildcard and Regex exclusions
sudo tee /etc/bareos/bareos-dir.d/fileset/EtcOnly.conf <<'EOF'
FileSet {
  Name = "EtcOnly"

  Include {
    Options {
      Signature = SHA1
      Compression = GZIP
      No Atime = yes
      XAttr = yes
      ACL Support = yes
    }

    File = /etc
  }

  Exclude {
    # Pattern-based exclusion using Options block within Exclude
    Options {
      # Wildcard match (case-sensitive)
      WildFile = "*.tmp"
      WildFile = "*.swp"
      WildFile = "*~"

      # Regex match
      # RegexFile = ".*\.pyc$"

      # Exclude directories matching pattern
      WildDir = "/etc/alternatives"

      Exclude = yes
    }
  }
}
EOF
```

**FileSet Options directives:**

| Directive | Description |
|-----------|-------------|
| `Signature = MD5\|SHA1\|SHA256` | File checksum algorithm |
| `Compression = GZIP\|LZ4` | Enable compression |
| `One FS = yes` | Don't cross filesystem boundaries |
| `No Atime = yes` | Restore atime after reading (don't update it) |
| `Sparse = yes` | Handle sparse files efficiently |
| `XAttr = yes` | Back up extended attributes |
| `ACL Support = yes` | Back up POSIX ACLs |
| `Checkfile Changes = yes` | Use checksum to detect changes (slower, more accurate) |
| `HardLinks = yes` | Preserve hard link relationships |
| `Wild = PATTERN` | Wildcard-exclude/include |
| `Regex = PATTERN` | Regex-exclude/include |
| `Exclude = yes` | Makes this Options block an exclusion rule |

### 4.7 Pool resource (`bareos-dir.d/pool/`)

Pools group Volumes (storage units) and define retention policies.

```bash
sudo tee /etc/bareos/bareos-dir.d/pool/Full.conf <<'EOF'
Pool {
  Name = Full

  # Type: Backup, Archive, Cloned, Migration, Scratch, etc.
  Pool Type = Backup

  # Recycling and lifecycle
  # After all jobs in a volume expire, can the volume be reused?
  Recycle = yes

  # Automatically purge expired volumes
  Auto Prune = yes

  # How long to keep Full backup volumes before recycling
  Volume Retention = 12 months

  # Limit on volume size (0 = unlimited)
  Maximum Volume Bytes = 0

  # Maximum number of jobs per volume (0 = unlimited)
  Maximum Volume Jobs = 1

  # Allow only one session per volume (cleaner, one job per volume)
  # Maximum Volume Jobs = 1  is best for Full backups

  # Label format for new volumes
  # Bareos creates volumes named: Full-0001, Full-0002, ...
  Label Format = "Full-"

  # Maximum volumes in this pool (0 = unlimited)
  Maximum Volumes = 100

  # After purge, what to do with the volume?
  # Truncate = reduces volume file to zero after purge
  Action On Purge = Truncate
}
EOF

sudo tee /etc/bareos/bareos-dir.d/pool/Incremental.conf <<'EOF'
Pool {
  Name = Incremental
  Pool Type = Backup
  Recycle = yes
  Auto Prune = yes
  Volume Retention = 30 days    # Shorter retention for incrementals
  Maximum Volume Jobs = 10       # Multiple jobs per volume (incrementals are small)
  Label Format = "Incr-"
  Maximum Volumes = 200
  Action On Purge = Truncate
}
EOF

sudo tee /etc/bareos/bareos-dir.d/pool/Differential.conf <<'EOF'
Pool {
  Name = Differential
  Pool Type = Backup
  Recycle = yes
  Auto Prune = yes
  Volume Retention = 60 days
  Maximum Volume Jobs = 5
  Label Format = "Diff-"
  Maximum Volumes = 100
  Action On Purge = Truncate
}
EOF
```

**Volume lifecycle:**
```
New → Append (being written) → Full (no more space/jobs) → Used
→ Purged (all jobs expired) → Recycled (ready to overwrite)
```

### 4.8 Storage resource (`bareos-dir.d/storage/File.conf`)

The Storage resource in the Director points to a Storage Daemon device:

```bash
sudo tee /etc/bareos/bareos-dir.d/storage/File.conf <<'EOF'
Storage {
  Name = FileStorage

  # Hostname/IP of the Storage Daemon
  # Use "localhost" if SD is on the same machine as Director
  Address = localhost

  # Port the Storage Daemon listens on
  SD Port = 9103

  # Password used to authenticate Director to Storage Daemon
  # Must match the password in bareos-sd.d/director/bareos-dir.conf
  Password = "SD_PASSWORD_CHANGE_ME"

  # Name of the Device in the Storage Daemon config
  Device = FileStorage

  # Media type — must match the Device's Media Type in SD config
  Media Type = File

  # Allow multiple concurrent jobs on this storage
  Maximum Concurrent Jobs = 10
}
EOF
```

### 4.9 Client resource (`bareos-dir.d/client/bareos-fd.conf`)

A Client resource defines a File Daemon (client) that this Director can connect to:

```bash
# The local File Daemon (already created at install)
sudo tee /etc/bareos/bareos-dir.d/client/bareos-fd.conf <<'EOF'
Client {
  Name = bareos-fd

  # Hostname or IP of the client machine
  Address = localhost

  # Port the File Daemon listens on
  FD Port = 9102

  # Catalog to use for this client's metadata
  Catalog = MyCatalog

  # Password to authenticate Director to File Daemon
  # Must match in bareos-fd.d/director/bareos-dir.conf
  Password = "FD_PASSWORD_CHANGE_ME"

  # How long to keep job records for this client
  Job Retention = 6 months

  # How long to keep file records (per-file metadata)
  File Retention = 60 days

  # Auto-prune expired records
  Auto Prune = yes

  # Maximum concurrent jobs from this client
  Maximum Concurrent Jobs = 5
}
EOF
```

### 4.10 Job resource (`bareos-dir.d/job/`)

The Job resource ties everything together:

```bash
sudo tee /etc/bareos/bareos-dir.d/job/BackupLocalServer.conf <<'EOF'
Job {
  Name = "BackupLocalServer"

  # Inherit defaults from JobDefs
  JobDefs = "DefaultJob"

  # Which client to back up
  Client = bareos-fd

  # Which FileSet to use
  FileSet = "DefaultFileSet"

  # Override the schedule from JobDefs
  Schedule = "WeeklyCycle"

  # Pool overrides (optional — takes precedence over JobDefs)
  Full Backup Pool = Full
  Incremental Backup Pool = Incremental
  Differential Backup Pool = Differential

  # Run scripts before/after the job on the Director
  # Run Before Job = "/usr/local/bin/pre-backup.sh"
  # Run After Job = "/usr/local/bin/post-backup.sh"
}
EOF

# Separate job to back up the Bareos catalog itself
sudo tee /etc/bareos/bareos-dir.d/job/BackupCatalog.conf <<'EOF'
Job {
  Name = "BackupCatalog"
  JobDefs = "DefaultJob"
  Level = Full
  FileSet = "Catalog"
  Schedule = "WeeklyCycleAfterBackup"
  Client = bareos-fd
  Storage = FileStorage
  Pool = Full
  Messages = Standard
  Catalog = MyCatalog
  # Run catalog backup script before job
  Run Before Job = "/usr/lib/bareos/scripts/make_catalog_backup.pl MyCatalog"
  # Clean up after catalog backup
  Run After Job = "/usr/lib/bareos/scripts/delete_catalog_backup"
  Write Bootstrap = "/var/lib/bareos/%n.bsr"
  Priority = 11   # Run after regular backup jobs
}
EOF
```

[↑ Table of Contents](#table-of-contents)

---

## 5. Storage Daemon Configuration

### 5.1 Storage Daemon self-definition (`bareos-sd.d/storage/bareos-sd.conf`)

```bash
sudo tee /etc/bareos/bareos-sd.d/storage/bareos-sd.conf <<'EOF'
Storage {
  Name = bareos-sd

  # Port this SD listens on
  SD Port = 9103

  # Maximum concurrent jobs
  Maximum Concurrent Jobs = 20

  # Working directory for temporary files
  Working Directory = /var/lib/bareos

  # PID file location
  PID Directory = /var/run

  # Heartbeat interval to detect dead connections
  Heartbeat Interval = 120

  # TLS (covered in Section 14)
}
EOF
```

### 5.2 Authorise the Director to connect (`bareos-sd.d/director/bareos-dir.conf`)

```bash
sudo tee /etc/bareos/bareos-sd.d/director/bareos-dir.conf <<'EOF'
Director {
  # This must match the Director's Name
  Name = bareos-dir

  # Password the Director uses to authenticate to this SD
  # Must match the Password in Director's Storage resource
  Password = "SD_PASSWORD_CHANGE_ME"

  # Monitor-only access (set to yes for monitoring only)
  Monitor = no
}
EOF
```

### 5.3 Device resource — disk-based storage (`bareos-sd.d/device/FileStorage.conf`)

```bash
# Create the storage directory
sudo mkdir -p /backup/bareos
sudo chown bareos:bareos /backup/bareos
sudo chmod 750 /backup/bareos

sudo tee /etc/bareos/bareos-sd.d/device/FileStorage.conf <<'EOF'
Device {
  Name = FileStorage

  # Device type: File (disk), Tape, FIFO, etc.
  Media Type = File

  # Path where backup data is stored
  Archive Device = /backup/bareos

  # Label media automatically if unlabelled
  Label Media = yes

  # Auto-mount device (yes for disk, no for tape)
  Random Access = yes

  # Auto-mount when needed
  AutoMount = yes

  # Allow multiple volumes to be opened simultaneously
  AutoChanger = no

  # Remove volume file on close (no = keep the file)
  # Useful for cleaning up disk files
  # AlwaysOpen = no

  # Maximum open wait time
  Maximum Open Wait = 300

  # Volume Poll Interval (for tape changers)
  # Volume Poll Interval = 360
}

# Second device for incremental backups (separate directory)
Device {
  Name = FileStorageIncremental
  Media Type = File
  Archive Device = /backup/bareos-incr
  Label Media = yes
  Random Access = yes
  AutoMount = yes
}
EOF

# Create the incremental storage directory
sudo mkdir -p /backup/bareos-incr
sudo chown bareos:bareos /backup/bareos-incr
```

[↑ Table of Contents](#table-of-contents)

---

## 6. File Daemon Configuration

### 6.1 File Daemon self-definition (`bareos-fd.d/client/myself.conf`)

```bash
sudo tee /etc/bareos/bareos-fd.d/client/myself.conf <<'EOF'
Client {
  Name = bareos-fd

  # Port this FD listens on
  FD Port = 9102

  # Working directory
  Working Directory = /var/lib/bareos

  # PID file location
  PID Directory = /var/run

  # Maximum concurrent jobs this FD will process
  Maximum Concurrent Jobs = 20

  # Heartbeat interval
  Heartbeat Interval = 120

  # Buffer size for data transfer
  # Larger buffers can improve performance on fast networks
  # Maximum Network Buffer Size = 65536
}
EOF
```

### 6.2 Authorise the Director to connect (`bareos-fd.d/director/bareos-dir.conf`)

```bash
sudo tee /etc/bareos/bareos-fd.d/director/bareos-dir.conf <<'EOF'
Director {
  # This must match the Director's Name exactly
  Name = bareos-dir

  # Password the Director uses to authenticate to this FD
  # Must match the Password in Director's Client resource
  Password = "FD_PASSWORD_CHANGE_ME"

  # Allow this Director to monitor this FD (read-only)
  Monitor = no

  # Allow this Director to restore files to this client
  # (required for restores)
  # Restrict access to specific paths (optional)
  # Allowed Job Command = backup
  # Allowed Job Command = restore
  # Allowed Job Command = verify
}
EOF
```

[↑ Table of Contents](#table-of-contents)

---

## 7. Setting Passwords

Bareos uses random passwords generated during installation. For a new setup, generate strong passwords and set them consistently:

```bash
# Generate strong random passwords
DIRECTOR_PASSWORD=$(openssl rand -base64 32)
SD_PASSWORD=$(openssl rand -base64 32)
FD_PASSWORD=$(openssl rand -base64 32)
CONSOLE_PASSWORD=$(openssl rand -base64 32)

echo "Director password: $DIRECTOR_PASSWORD"
echo "SD password:       $SD_PASSWORD"
echo "FD password:       $FD_PASSWORD"
echo "Console password:  $CONSOLE_PASSWORD"

# Now set them in the appropriate config files:
# SD_PASSWORD → Director/storage/File.conf AND bareos-sd.d/director/bareos-dir.conf
# FD_PASSWORD → Director/client/bareos-fd.conf AND bareos-fd.d/director/bareos-dir.conf
# CONSOLE_PASSWORD → bareos-dir.d/console/bareos-mon.conf AND bconsole.conf
```

**Consistency rule:** Each password must appear in exactly two places:
- SD password: Director's Storage resource + SD's Director resource
- FD password: Director's Client resource + FD's Director resource

[↑ Table of Contents](#table-of-contents)

---

## 8. bconsole Configuration

```bash
sudo tee /etc/bareos/bconsole.conf <<'EOF'
Director {
  # Address of the Director
  Name = bareos-dir
  Address = localhost
  DIRport = 9101

  # Password for bconsole authentication
  Password = "CONSOLE_PASSWORD_CHANGE_ME"
}
EOF
```

[↑ Table of Contents](#table-of-contents)

---

## 9. Validate and Reload Configuration

```bash
# Test Director config for syntax errors
sudo bareos-dir -t

# Test Storage Daemon config
sudo bareos-sd -t

# Test File Daemon config
sudo bareos-fd -t

# If all pass, restart services
sudo systemctl restart bareos-dir bareos-sd bareos-fd

# Check all started successfully
sudo systemctl status bareos-dir bareos-sd bareos-fd

# Watch logs for errors
sudo journalctl -fu bareos-dir
```

[↑ Table of Contents](#table-of-contents)

---

## 10. bconsole — Complete Command Reference

### 10.1 Connecting

```bash
# Connect to Director as root
sudo bconsole

# You should see:
# Connecting to Director localhost:9101
# 1000 OK: bareos-dir Version: ...
# Enter a period to cancel a command.
# *
```

### 10.2 Status commands

```bash
# Show Director status
*status dir

# Show Storage Daemon status
*status storage=FileStorage

# Show File Daemon status on all clients
*status client=bareos-fd

# Show all running jobs
*status all

# Show current jobs in the job queue
*list jobs

# Show running jobs
*list running jobs
```

### 10.3 Running jobs manually

```bash
# Run a job interactively (prompts for options)
*run

# Run a specific job
*run job=BackupLocalServer

# Run with a specific level override
*run job=BackupLocalServer level=Full

# Run immediately (no confirmation)
*run job=BackupLocalServer level=Full yes

# Estimate how many files/bytes a job would process
*estimate job=BackupLocalServer level=Full

# Run with a specific client
*run job=BackupLocalServer client=backup-client-1
```

### 10.4 Monitoring jobs

```bash
# List recent jobs
*list jobs

# List last 20 jobs
*list jobs last=20

# Show details of a specific job
*show job=BackupLocalServer

# View messages in console
*messages

# Watch job output in real time (tail-like)
# In the job confirmation, say yes; then watch console messages
```

### 10.5 Listing volumes and files

```bash
# List all volumes
*list volumes

# List volumes in a specific pool
*list volumes pool=Full

# List files in a specific job
*list files jobid=1

# List files in a jobid with a path filter
*list files jobid=1

# Find which job backed up a specific file
*llist files job=BackupLocalServer
```

### 10.6 Volume management

```bash
# Label a new volume (create a new volume in a pool)
*label storage=FileStorage pool=Full volume=Full-0001

# Update a volume's settings
*update volume=Full-0001 volstatus=Append

# Purge a volume (mark all jobs as expired, allow reuse)
*purge volume=Full-0001

# Truncate a purged volume file (free disk space)
*truncate volume=Full-0001

# Move a volume between pools
*update volume=Full-0001 pool=Incremental

# List volume details
*llist volume=Full-0001
```

### 10.7 Pruning the catalog

```bash
# Prune expired job records from catalog
*prune jobs client=bareos-fd

# Prune expired file records (frees significant catalog space)
*prune files client=bareos-fd

# Prune all expired volumes
*prune volumes

# Prune everything for a client
*prune all client=bareos-fd yes
```

### 10.8 Cancelling jobs

```bash
# List running jobs to find the JobId
*status dir

# Cancel a specific job
*cancel jobid=5

# Cancel all running jobs
*cancel all yes
```

### 10.9 Show config resources

```bash
# Show all jobs defined
*show jobs

# Show all pools
*show pools

# Show all clients
*show clients

# Show all filesets
*show filesets

# Show all schedules
*show schedules

# Show all storage resources
*show storages

# Show everything
*show all
```

[↑ Table of Contents](#table-of-contents)

---

## 11. The Interactive Restore Wizard

The restore wizard is the primary way to restore files using bconsole.

```
*restore
```

Bareos shows a menu:
```
To select the JobIds, you have the following choices:
     1: List last 20 Jobs run
     2: List Jobs where a given File is saved
     3: Enter list of comma separated JobIds to select
     4: Enter SQL list command
     5: Select the most recent backup for a client
     6: Select backup for a client before a specified time
     7: Enter a list of files to restore
     8: Enter a list of files to restore before a specified time
     9: Find the JobIds of the most recent backup for a client
    10: Find the JobIds for a backup for a client before a specified time
    11: Enter a list of directories to restore for found JobIds
    12: Select full restore to a specified Job date
    13: Cancel
```

### Restore the latest backup of a client

```
Select item: 5
Defined Clients:
     1: bareos-fd
     2: backup-client-1
Select the Client: 1

The defined FileSet resources are:
     1: DefaultFileSet
Select FileSet resource: 1

+-------+-------+----------+------------+---------+------+-------+
| JobId | Level | JobFiles | StartTime  | VolumeName | ... |
+-------+-------+----------+------------+---------+------+-------+
|     1 | F     |    12345 | 2026-02-17 | Full-0001  | ... |
|     2 | I     |      234 | 2026-02-18 | Incr-0001  | ... |
|     3 | I     |       89 | 2026-02-19 | Incr-0001  | ... |
+-------+-------+----------+------------+---------+------+-------+

Automatically selected JobId=3
Building directory tree for JobId(s) 1,2,3 ...
12345 files inserted into the tree.

You are now entering file selection mode where you add (mark) and
remove (unmark) files to be restored.

cwd is: /
$ ls
  boot/  etc/  home/  opt/  root/  srv/  usr/  var/
$ cd etc
$ ls
  hostname  nginx/  ssh/  ...
$ add ssh/sshd_config        ← mark file for restore
$ add nginx/                 ← mark entire directory
$ done                       ← or: extract
```

After selecting files, Bareos asks:
```
Bootstrap records written to /var/lib/bareos/bareos-dir.restore.1.bsr

The Job will require the following
   Volume(s)                 Storage(s)                SD Device(s)
===========================================================================
   Full-0001                 FileStorage               FileStorage
   Incr-0001                 FileStorage               FileStorage

Defined Clients:
     1: bareos-fd
     2: backup-client-1

Using Client "bareos-fd"

Where to restore to (/, enter)? /restore/
Replace (never): never
```

### Restore to a specific date

```
Select item: 6
Enter date/time in the form YYYY-MM-DD HH:MM:SS: 2026-02-18 12:00:00
[continues with file browser...]
```

### Restore all files from the latest backup

```
*restore client=bareos-fd where=/ all yes
```

[↑ Table of Contents](#table-of-contents)

---

## 12. Adding a Remote Client

### 12.1 Install File Daemon on the client

```bash
# On backup-client:
sudo dnf install -y epel-release

# Add Bareos repo (same as on server)
wget -O /etc/yum.repos.d/bareos.repo \
  https://download.bareos.org/current/EL_9/bareos.repo

sudo dnf install -y bareos-filedaemon

# Generate a password for this client
CLIENT_PASSWORD=$(openssl rand -base64 32)
echo "Client password: $CLIENT_PASSWORD"
```

### 12.2 Configure File Daemon on the client

```bash
# On backup-client: /etc/bareos/bareos-fd.d/client/myself.conf
sudo tee /etc/bareos/bareos-fd.d/client/myself.conf <<'EOF'
Client {
  Name = backup-client-1     # Must be unique in the Bareos environment
  FD Port = 9102
  Working Directory = /var/lib/bareos
  PID Directory = /var/run
  Maximum Concurrent Jobs = 5
  Heartbeat Interval = 120
}
EOF

# Authorise the Director to connect to this FD
sudo tee /etc/bareos/bareos-fd.d/director/bareos-dir.conf <<'EOF'
Director {
  Name = bareos-dir                  # Must match Director's Name
  Password = "CLIENT1_PASSWORD_HERE" # Same password used in Director's Client resource
  Monitor = no
}
EOF

# Start and enable the File Daemon on the client
sudo systemctl enable --now bareos-fd

# Open firewall on client
sudo firewall-cmd --permanent --add-port=9102/tcp
sudo firewall-cmd --reload
```

### 12.3 Add the client to the Director

```bash
# On backup-server (Director):
sudo tee /etc/bareos/bareos-dir.d/client/backup-client-1.conf <<'EOF'
Client {
  Name = backup-client-1
  Address = 192.168.100.20     # Client's IP or hostname
  FD Port = 9102
  Catalog = MyCatalog
  Password = "CLIENT1_PASSWORD_HERE"    # Same as in client's FD director resource
  Job Retention = 6 months
  File Retention = 60 days
  Auto Prune = yes
  Maximum Concurrent Jobs = 5
}
EOF

# Create a job for the new client
sudo tee /etc/bareos/bareos-dir.d/job/BackupClient1.conf <<'EOF'
Job {
  Name = "BackupClient1"
  JobDefs = "DefaultJob"
  Client = backup-client-1
  FileSet = "DefaultFileSet"
  Schedule = "WeeklyCycle"
  Full Backup Pool = Full
  Incremental Backup Pool = Incremental
}
EOF

# Reload Director config
sudo bareos-dir -t && sudo systemctl reload bareos-dir

# Test connectivity from bconsole
sudo bconsole
*status client=backup-client-1
```

[↑ Table of Contents](#table-of-contents)

---

## 13. Database Maintenance

### 13.1 Automated catalog backup

```bash
# The BackupCatalog job (created in Section 4.10) handles this
# Manually trigger:
sudo bconsole
*run job=BackupCatalog yes
```

### 13.2 Manual catalog backup

```bash
# Dump the Bareos catalog database
sudo -u bareos /usr/lib/bareos/scripts/make_catalog_backup.pl MyCatalog

# The dump is placed in /var/lib/bareos/bareos.sql
ls -lh /var/lib/bareos/*.sql
```

### 13.3 Catalog database check and repair

```bash
# Check catalog for consistency
sudo bareos-dbcheck -B MyCatalog

# Interactive mode — check and fix
sudo bareos-dbcheck -B MyCatalog -c /etc/bareos/bareos-dir.d/catalog/MyCatalog.conf
```

### 13.4 Catalog size management

The catalog grows over time as file records accumulate. Monitor it:

```bash
# Check catalog database size
sudo mysql -u root -e "SELECT table_name, \
  ROUND(data_length/1024/1024,2) AS 'Data MB', \
  ROUND(index_length/1024/1024,2) AS 'Index MB' \
  FROM information_schema.tables \
  WHERE table_schema='bareos' \
  ORDER BY data_length DESC;"

# Key tables:
# File — grows large if File Retention is long
# Job  — one row per job
# Path — directory paths

# The biggest space saver: reduce File Retention in JobDefs
# Set File Retention to 30 days (less browseable history, smaller catalog)
```

### 13.5 Pruning the catalog from bconsole

```bash
sudo bconsole
# Prune all expired file records for all clients
*prune files all yes

# Prune all expired job records
*prune jobs all yes

# Prune expired volumes
*prune volumes all yes
```

[↑ Table of Contents](#table-of-contents)

---

## 14. TLS Configuration

Bareos supports TLS 1.2+ between all daemons. This section configures mutual TLS.

### 14.1 Generate TLS certificates

```bash
# Create a CA and certificates for each daemon
sudo mkdir -p /etc/bareos/tls

# Create CA key and certificate
sudo openssl req -newkey rsa:4096 -nodes -x509 \
  -keyout /etc/bareos/tls/ca.key \
  -out /etc/bareos/tls/ca.crt \
  -days 3650 \
  -subj "/CN=Bareos-CA/O=Bareos"

# Create Director certificate
sudo openssl req -newkey rsa:4096 -nodes \
  -keyout /etc/bareos/tls/director.key \
  -out /etc/bareos/tls/director.csr \
  -subj "/CN=bareos-dir"
sudo openssl x509 -req \
  -in /etc/bareos/tls/director.csr \
  -CA /etc/bareos/tls/ca.crt \
  -CAkey /etc/bareos/tls/ca.key \
  -CAcreateserial \
  -out /etc/bareos/tls/director.crt \
  -days 1825

# Create Storage Daemon certificate
sudo openssl req -newkey rsa:4096 -nodes \
  -keyout /etc/bareos/tls/storage.key \
  -out /etc/bareos/tls/storage.csr \
  -subj "/CN=bareos-sd"
sudo openssl x509 -req \
  -in /etc/bareos/tls/storage.csr \
  -CA /etc/bareos/tls/ca.crt \
  -CAkey /etc/bareos/tls/ca.key \
  -CAcreateserial \
  -out /etc/bareos/tls/storage.crt \
  -days 1825

# Create File Daemon certificate
sudo openssl req -newkey rsa:4096 -nodes \
  -keyout /etc/bareos/tls/fd.key \
  -out /etc/bareos/tls/fd.csr \
  -subj "/CN=bareos-fd"
sudo openssl x509 -req \
  -in /etc/bareos/tls/fd.csr \
  -CA /etc/bareos/tls/ca.crt \
  -CAkey /etc/bareos/tls/ca.key \
  -CAcreateserial \
  -out /etc/bareos/tls/fd.crt \
  -days 1825

# Set permissions
sudo chown -R bareos:bareos /etc/bareos/tls
sudo chmod 640 /etc/bareos/tls/*.key
sudo chmod 644 /etc/bareos/tls/*.crt
```

### 14.2 Enable TLS in Director

Add TLS block to `bareos-dir.d/director/bareos-dir.conf`:

```bash
# Edit the Director resource to add TLS
sudo tee -a /etc/bareos/bareos-dir.d/director/bareos-dir.conf <<'EOF'

# Append to existing Director {} block — edit manually to merge:
# TLS Enable = yes
# TLS Require = yes
# TLS Verify Peer = yes
# TLS CA Certificate File = /etc/bareos/tls/ca.crt
# TLS Certificate = /etc/bareos/tls/director.crt
# TLS Key = /etc/bareos/tls/director.key
EOF
```

### 14.3 Enable TLS in Storage Daemon

Add to `bareos-sd.d/storage/bareos-sd.conf`:

```
TLS Enable = yes
TLS Require = yes
TLS Verify Peer = yes
TLS CA Certificate File = /etc/bareos/tls/ca.crt
TLS Certificate = /etc/bareos/tls/storage.crt
TLS Key = /etc/bareos/tls/storage.key
```

### 14.4 Enable TLS in File Daemon

Add to `bareos-fd.d/client/myself.conf`:

```
TLS Enable = yes
TLS Require = yes
TLS Verify Peer = yes
TLS CA Certificate File = /etc/bareos/tls/ca.crt
TLS Certificate = /etc/bareos/tls/fd.crt
TLS Key = /etc/bareos/tls/fd.key
```

### 14.5 Verify TLS is active

```bash
# After restarting all daemons, verify TLS in status output
sudo bconsole
*status dir
# Look for: "TLS: enabled"
```

[↑ Table of Contents](#table-of-contents)

---

## 15. Monitoring and Alerting

### 15.1 Email notifications

Configure email in the Messages resource (Section 4.3):

```bash
# Install mail sending capability
sudo dnf install -y postfix
sudo systemctl enable --now postfix

# Or use bsmtp (Bareos simple SMTP)
# Edit /etc/bareos/bareos-dir.d/messages/Standard.conf
# MailCommand = "/usr/sbin/bsmtp -h smtp.yourserver.com -f bareos@yourserver.com ..."
```

### 15.2 systemd journal monitoring

```bash
# Watch Director log in real time
sudo journalctl -fu bareos-dir

# Check for errors in the last hour
sudo journalctl -u bareos-dir --since "1 hour ago" | grep -i "error\|warn\|fail"

# See all job termination messages
sudo journalctl -u bareos-dir | grep "Job.*OK\|Job.*Error\|Termination"
```

### 15.3 Bareos log file

```bash
# View the Bareos log
sudo tail -f /var/log/bareos/bareos.log

# Find failed jobs
sudo grep -i "error\|failed\|warn" /var/log/bareos/bareos.log | tail -50
```

### 15.4 Monitor from bconsole

```bash
sudo bconsole
# Summary of recent jobs
*list jobs last=7

# Jobs that failed
*list jobs status=E

# Jobs that terminated with warnings
*list jobs status=W
```

[↑ Table of Contents](#table-of-contents)

---

## 16. bextract — Restore Without a Running Director

`bextract` reads Bareos volumes directly without needing the Director or catalog. This is the tool for **bare-metal emergency restores**.

```bash
# List contents of a volume
sudo bls \
  -V Full-0001 \
  FileStorage

# Extract all files from a volume to /restore
sudo bextract \
  -V Full-0001 \
  FileStorage \
  /restore/

# Extract a specific directory
sudo bextract \
  -V Full-0001 \
  -r /etc/nginx \
  FileStorage \
  /restore/

# Using a Bootstrap file (generated during restore planning)
sudo bextract \
  -b /var/lib/bareos/bareos-dir.restore.1.bsr \
  FileStorage \
  /restore/
```

**Note:** `bextract` needs to know the storage device path. For disk-based storage (`File` device), the archive device is the directory configured in the SD device resource (e.g., `/backup/bareos`).

[↑ Table of Contents](#table-of-contents)

---

## 17. Bareos WebUI

The Bareos WebUI is a PHP web application that communicates with the Bareos Director over the same bconsole protocol (TCP 9101). It provides a browser-based dashboard for monitoring jobs, browsing backed-up file trees, triggering runs, and performing point-and-click restores — all without touching bconsole.

### 17.1 Architecture

```
Browser  ──HTTPS──►  Apache httpd  ──PHP-FPM──►  bareos-webui (PHP/ZF2)
                                                         │
                                                    TCP 9101
                                                         │
                                               bareos-dir (Director)
```

Key points:
- The WebUI does **not** need to run on the same host as the Director
- It authenticates to the Director using a **named Console resource** (not the root console)
- Multiple Directors can be configured in a single WebUI instance
- PHP-FPM is required (Apache `mod_php` was dropped in Bareos 22+)
- The WebUI has no user database of its own — every login maps to a Director Console resource

### 17.2 Features

| Feature | Notes |
|---------|-------|
| Dashboard | Last 24 h job summary, running jobs, failed jobs |
| Job management | Run, rerun, cancel, enable/disable jobs and schedules |
| Client management | Enable/disable clients, view status |
| File-tree restore | Browse snapshot of any job; restore individual files or whole trees |
| Volume management | View pool/volume status; label new media |
| Tape autochanger | Import/export, slot status (if applicable) |
| bconsole tab | Non-interactive commands only |
| Multi-director | Switch between multiple Directors from the same UI |
| ACL enforcement | Per-user access profiles via Director Console/Profile resources |

### 17.3 Install bareos-webui

The `bareos-webui` package is in the same Bareos repository added in Section 2.

```bash
# Install WebUI, Apache, PHP-FPM and required modules
dnf install -y bareos-webui httpd php-fpm mod_fcgid

# The bareos-webui package drops an Apache config automatically:
ls /etc/httpd/conf.d/bareos-webui.conf

# Enable and start services
systemctl enable --now php-fpm
systemctl enable --now httpd

# Verify both are running
systemctl status php-fpm httpd
```

#### Firewall

```bash
# Open HTTP (and HTTPS if you add TLS to Apache later)
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

### 17.4 SELinux — Required Boolean

On RHEL 10 with SELinux enforcing, Apache must be permitted to make outbound TCP connections to port 9101 (the Director). Without this, every login attempt fails silently.

```bash
# Allow Apache (httpd) to connect to the Director
setsebool -P httpd_can_network_connect on

# Verify
getsebool httpd_can_network_connect
# httpd_can_network_connect --> on
```

If the WebUI and Director are on **different hosts**, this boolean is essential. If they are on the **same host** and you use `localhost`, it is still required because the network stack is still used.

### 17.5 Apache Configuration

The package places a ready-to-use vhost fragment at `/etc/httpd/conf.d/bareos-webui.conf`. Inspect and optionally adjust it:

```bash
cat /etc/httpd/conf.d/bareos-webui.conf
```

Default content (annotated):

```apache
#
# Bareos WebUI Apache configuration
# Installed by: bareos-webui package
#

Alias /bareos-webui /usr/share/bareos-webui/public

# PHP-FPM proxy — forward .php requests to the FPM socket
<FilesMatch "\.php$">
    SetHandler "proxy:unix:/run/php-fpm/www.sock|fcgi://localhost"
</FilesMatch>

<Directory /usr/share/bareos-webui/public>
    Options None
    AllowOverride None
    # mod_rewrite: required for clean URL routing in the ZF2 framework
    RewriteEngine On
    RewriteCond %{REQUEST_FILENAME} -s [OR]
    RewriteCond %{REQUEST_FILENAME} -l [OR]
    RewriteCond %{REQUEST_FILENAME} -d
    RewriteRule ^.*$ - [NC,L]
    RewriteRule ^.*$ index.php [NC,L]
    Require all granted
</Directory>
```

After any Apache config change:

```bash
# Test config syntax
httpd -t

# Reload (no downtime)
systemctl reload httpd
```

#### php-fpm pool timezone

The WebUI displays timestamps. Align the timezone with your Director:

```bash
# Find the active php-fpm pool config
ls /etc/php-fpm.d/
# Typically: /etc/php-fpm.d/www.conf

# Add or ensure this line is set in /etc/php.ini or /etc/php-fpm.d/www.conf
grep -n "date.timezone" /etc/php.ini

# Set it:
sed -i 's/^;date.timezone.*/date.timezone = Europe\/London/' /etc/php.ini
# Replace Europe/London with your actual timezone (e.g. America/New_York, UTC)

systemctl restart php-fpm
```

### 17.6 Director: Console Resource for WebUI

The WebUI logs in as a named Console. You must create at least one Console resource in the Director and assign it a Profile that grants the required permissions.

#### Method A — bconsole (recommended, live config)

```bash
sudo bconsole
*reload
*configure add console name=admin password=StrongP@ssw0rd profile=webui-admin tlsenable=false
*reload
*quit
```

#### Method B — drop-in config file

```bash
cat > /etc/bareos/bareos-dir.d/console/webui-admin.conf << 'EOF'
Console {
  Name        = "admin"            # WebUI login username
  Password    = "StrongP@ssw0rd"   # WebUI login password — change this
  Profile     = "webui-admin"      # References a Profile resource (see 17.7)
  TlsEnable   = false              # TLS between WebUI and Director is PSK-only;
                                   # for cert-based TLS see Section 17.11
}
EOF

# Validate and reload
bareos-dir -t && systemctl reload bareos-dir
```

### 17.7 Director: Profile Resources

A Profile defines which bconsole commands and which resources a Console can access. The `bareos-webui` package ships `webui-admin.conf` in `/etc/bareos/bareos-dir.d/profile/` — but you should understand and optionally extend it.

#### Full admin profile (ships with package)

```bash
cat /etc/bareos/bareos-dir.d/profile/webui-admin.conf
```

```
Profile {
  Name = "webui-admin"

  # CommandACL: what bconsole commands this profile may issue.
  # !command = deny; *all* = allow everything not explicitly denied.
  # The denied commands below prevent destructive or dangerous operations
  # through the web interface even for admins.
  CommandACL = !.bvfs_clear_cache   # prevent accidental cache wipe
  CommandACL = !.exit               # cannot exit the Director process
  CommandACL = !.sql                # no raw SQL queries
  CommandACL = !configure           # no live config changes via WebUI
  CommandACL = !create              # no catalog record creation
  CommandACL = !delete              # no hard deletes
  CommandACL = !purge               # no volume purge
  CommandACL = !sqlquery            # no raw SQL
  CommandACL = !umount, !unmount    # no storage unmount
  CommandACL = *all*                # allow everything else

  JobACL      = *all*               # access to all jobs
  ScheduleACL = *all*               # access to all schedules
  CatalogACL  = *all*               # access to all catalogs
  PoolACL     = *all*               # access to all pools
  StorageACL  = *all*               # access to all storage resources
  ClientACL   = *all*               # access to all clients
  FileSetACL  = *all*               # access to all filesets
  WhereACL    = *all*               # can restore to any path
  PluginOptionsACL = *all*          # can pass plugin options in restore form
}
```

#### Limited operator profile (run/restore specific jobs only)

```bash
cat > /etc/bareos/bareos-dir.d/profile/webui-operator.conf << 'EOF'
Profile {
  Name = "webui-operator"

  # Allow only the commands the operator needs
  CommandACL = .api, .help, use, version, status, show
  CommandACL = list, llist
  CommandACL = run, rerun, cancel, restore
  CommandACL = .clients, .jobs, .filesets, .pools, .storages, .defaults, .schedule
  CommandACL = .bvfs_update, .bvfs_get_jobids, .bvfs_lsdirs, .bvfs_lsfiles
  CommandACL = .bvfs_versions, .bvfs_restore, .bvfs_cleanup

  # Restrict to specific jobs only
  JobACL      = BackupLocalServer, BackupClient1, RestoreFiles
  ScheduleACL = WeeklyCycle
  CatalogACL  = MyCatalog
  PoolACL     = Full, Differential, Incremental
  StorageACL  = FileStorage
  ClientACL   = bareos-fd, backup-client-fd
  FileSetACL  = DefaultFileSet
  WhereACL    = *all*               # allow restore to any path
}
EOF
```

#### Read-only monitoring profile

```bash
cat > /etc/bareos/bareos-dir.d/profile/webui-readonly.conf << 'EOF'
Profile {
  Name = "webui-readonly"

  # Minimal command set: view only, no run/restore/modify
  CommandACL = .api, .help, use, version, status
  CommandACL = list, llist
  CommandACL = .clients, .jobs, .filesets, .pools, .storages, .defaults, .schedule
  # Bvfs commands needed only to browse the file tree (read-only restore form)
  CommandACL = .bvfs_lsdirs, .bvfs_lsfiles, .bvfs_update
  CommandACL = .bvfs_get_jobids, .bvfs_versions, .bvfs_restore

  JobACL      = *all*
  ScheduleACL = *all*
  CatalogACL  = *all*
  PoolACL     = *all*
  StorageACL  = *all*
  ClientACL   = *all*
  FileSetACL  = *all*
  WhereACL    = *all*
}
EOF
```

#### Create a read-only user

```bash
sudo bconsole
*configure add console name=monitor password=ReadOnlyP@ss profile=webui-readonly tlsenable=false
*reload
*quit
```

After creating or editing profiles and consoles, always reload:

```bash
bareos-dir -t && systemctl reload bareos-dir
```

### 17.8 `/etc/bareos-webui/directors.ini` — Full Annotated Config

This file tells the WebUI which Director(s) to connect to. Each `[section]` is one Director entry.

```ini
; /etc/bareos-webui/directors.ini
; One section per Director. The section name appears in the WebUI login dropdown.

;------------------------------------------------------------------------------
; Primary Director — local installation
;------------------------------------------------------------------------------
[local-director]

; Enable or disable this entry in the login dropdown.
; Set to "no" to temporarily hide without deleting.
enabled = "yes"

; IP address or FQDN of the Bareos Director.
; Use "localhost" or "127.0.0.1" when WebUI and Director are on the same host.
diraddress = "localhost"

; Director port. Default: 9101. Match the Port directive in bareos-dir.d/director/bareos-dir.conf
dirport = 9101

; If you have multiple catalogs, specify which one to use.
; Omit this line when you have a single catalog named MyCatalog.
;catalog = "MyCatalog"

; --- TLS settings ---
; TLS-PSK is NOT supported between WebUI and Director.
; Leave all TLS options false for plain TCP (use a firewall or SSH tunnel for security).
; For certificate-based TLS see Section 17.11.

; Whether to verify the Director's TLS certificate against the CA file below.
tls_verify_peer = false

; Whether the Director is configured to offer TLS.
server_can_do_tls = false

; Whether the Director requires TLS (will refuse non-TLS connections).
server_requires_tls = false

; Whether the WebUI (client side) offers TLS.
client_can_do_tls = false

; Whether the WebUI requires TLS.
client_requires_tls = false

; Path to the CA certificate file (PEM). Required when tls_verify_peer = true.
;ca_file = "/etc/bareos-webui/tls/BareosCA.crt"

; Path to client certificate + private key in a single PEM file.
; Used for mutual TLS (client cert auth).
;cert_file = "/etc/bareos-webui/tls/webui-console.pem"

; Passphrase to decrypt cert_file private key, if the key is encrypted.
;cert_file_passphrase = ""

; Common names the Director's certificate is allowed to present.
; Leave blank to accept any CN when tls_verify_peer = false.
;allowed_cns = ""

;------------------------------------------------------------------------------
; Secondary Director — remote backup server (disabled by default)
;------------------------------------------------------------------------------
[remote-director]
enabled      = "no"
diraddress   = "192.168.100.10"
dirport      = 9101
;catalog      = "MyCatalog"
tls_verify_peer    = false
server_can_do_tls  = false
server_requires_tls = false
client_can_do_tls  = false
client_requires_tls = false
```

### 17.9 `/etc/bareos-webui/configuration.ini` — Full Annotated Config

```ini
; /etc/bareos-webui/configuration.ini
; All values shown are the compiled-in defaults.
; Uncomment and change only what you need.

;------------------------------------------------------------------------------
; SESSION
;------------------------------------------------------------------------------
[session]
; How long (seconds) a WebUI session stays valid without activity.
; After this period the user is logged out automatically.
; Default: 3600 (1 hour). Reduce for higher-security environments.
timeout = 3600

;------------------------------------------------------------------------------
; DASHBOARD
;------------------------------------------------------------------------------
[dashboard]
; How often (milliseconds) the dashboard auto-refreshes job status.
; Default: 60000 (60 seconds). Set lower for real-time monitoring environments.
autorefresh_interval = 60000

;------------------------------------------------------------------------------
; TABLES
;------------------------------------------------------------------------------
[tables]
; Rows-per-page choices available in the dropdown on each table.
pagination_values = 10,25,50,100

; Default selected rows-per-page. Must be one of the pagination_values.
pagination_default_value = 25

; Persist table sort/filter/page state across page reloads (uses browser storage).
; Useful for operators who always want to see the same view.
save_previous_state = false

;------------------------------------------------------------------------------
; AUTOCHANGER / LABELLING
;------------------------------------------------------------------------------
[autochanger]
; Name of the Pool to use when labelling new blank media in the autochanger.
; Typically "Scratch" — media starts here and is moved to working pools.
;labelpooltype =

;------------------------------------------------------------------------------
; RESTORE
;------------------------------------------------------------------------------
[restore]
; Timeout (milliseconds) for loading the file tree in the restore module.
; Large backup jobs with millions of files may need this increased.
; Default: 120000 (2 minutes).
filetree_refresh_timeout = 120000

; When a client is selected, merge all job histories into one unified file tree.
; Set to false to pick a single backup job to browse.
; Default: true
merge_jobs = true

; When a client is selected, merge all FileSet definitions into one file tree.
; Default: true
merge_filesets = true

;------------------------------------------------------------------------------
; THEME
;------------------------------------------------------------------------------
[theme]
; Available themes: default, sunflower
; Default: sunflower
name = sunflower

;------------------------------------------------------------------------------
; EXPERIMENTAL (not for production use unless you know what you are doing)
;------------------------------------------------------------------------------
;[experimental]
; Show a graphical resource dependency map in the Director configuration view.
;configuration_resource_graph = false
```

After any change to `configuration.ini` or `directors.ini`:

```bash
# No service restart needed — the PHP app reads these files on every request.
# Clear your browser cache if old settings appear to persist.
```

### 17.10 Bvfs Cache — Keep the File-Tree Browser Responsive

The restore file-tree browser uses the **Bvfs** (Bareos Virtual Filesystem) API, which caches directory/file metadata from the catalog. For large jobs with millions of files, the cache must be pre-populated or browsing will time out.

#### Update cache after every backup job

Add a `RunScript` directive to each job that should be browsable:

```
# In bareos-dir.d/jobdefs/DefaultJob.conf — add inside the JobDefs block:
Run Script {
    Console    = ".bvfs_update jobid=%i"  # %i expands to the job's ID
    RunsWhen   = After                    # run after the backup job completes
    RunsOnClient = No                     # runs on Director, not the client
}
```

#### Update cache after catalog backup (catches all jobs at once)

```
# In bareos-dir.d/job/BackupCatalog.conf — add inside the Job block:
Run Script {
    Console      = ".bvfs_update"   # no jobid = update cache for ALL uncached jobs
    RunsWhen     = After
    RunsOnClient = No
}
```

#### Manual cache update from bconsole

```bash
sudo bconsole
*update bvfs jobid=all
*quit
```

### 17.11 TLS Between WebUI and Director (Certificate-Based)

TLS-PSK (used between Bareos daemons) is **not** supported for the WebUI connection. To encrypt WebUI ↔ Director traffic using certificates:

```bash
# 1. Copy the CA cert and create a PEM file for the WebUI console
#    (reuse the CA from Section 14 if already set up)

# Create a combined PEM: client cert + private key in one file
cat /etc/bareos/tls/bareos-webui.crt \
    /etc/bareos/tls/bareos-webui.key \
    > /etc/bareos-webui/tls/webui-console.pem
chmod 640 /etc/bareos-webui/tls/webui-console.pem
chown root:apache /etc/bareos-webui/tls/webui-console.pem

# Copy the CA cert
cp /etc/bareos/tls/BareosCA.crt /etc/bareos-webui/tls/BareosCA.crt

# 2. Update directors.ini for this director section:
#   tls_verify_peer     = true
#   server_can_do_tls   = true
#   server_requires_tls = true
#   client_can_do_tls   = true
#   client_requires_tls = true
#   ca_file             = "/etc/bareos-webui/tls/BareosCA.crt"
#   cert_file           = "/etc/bareos-webui/tls/webui-console.pem"

# 3. In the Director Console resource for the WebUI user:
#   TlsEnable   = yes
#   TlsRequire  = yes
#   TlsCaCertificateFile = /etc/bareos/tls/BareosCA.crt
#   TlsCertificate       = /etc/bareos/tls/bareos-webui.crt
#   TlsKey               = /etc/bareos/tls/bareos-webui.key

# 4. Reload Director
bareos-dir -t && systemctl reload bareos-dir

# 5. Restart Apache + php-fpm to pick up new ini values
systemctl restart php-fpm httpd
```

### 17.12 NGINX Alternative

If Apache is not desired, the WebUI runs equally well under NGINX + php-fpm:

```bash
dnf install -y nginx php-fpm

# /etc/nginx/conf.d/bareos-webui.conf
cat > /etc/nginx/conf.d/bareos-webui.conf << 'EOF'
server {
    listen      80;
    server_name backup-server;

    root  /usr/share/bareos-webui/public;
    index index.php;

    # Route all requests through the ZF2 front controller
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    # Pass PHP to php-fpm
    location ~ \.php$ {
        fastcgi_pass   unix:/run/php-fpm/www.sock;
        fastcgi_index  index.php;
        include        fastcgi_params;
        fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param  APPLICATION_ENV production;
    }

    # Deny access to hidden files
    location ~ /\. {
        deny all;
    }
}
EOF

nginx -t && systemctl enable --now nginx
systemctl restart php-fpm
```

Access at: `http://backup-server/` (root, not `/bareos-webui/`)

### 17.13 Multi-Director Setup

A single WebUI can monitor multiple independent Bareos Directors. This is useful when you have separate Bareos installations per environment (production, DR, staging).

Add each Director as a separate `[section]` in `directors.ini`:

```ini
[prod-director]
enabled    = "yes"
diraddress = "bareos-prod.example.com"
dirport    = 9101

[dr-director]
enabled    = "yes"
diraddress = "bareos-dr.example.com"
dirport    = 9101

[staging-director]
enabled    = "no"          ; hidden from login page but kept for future use
diraddress = "bareos-staging.example.com"
dirport    = 9101
```

On the login page the user sees a **Director** dropdown listing every enabled section. Credentials (Console name + password) must exist in the chosen Director's config.

> **Security note:** Each Director must independently have the Console resource configured with the matching password. There is no central credential store.

### 17.14 Accessing the WebUI

```
http://backup-server/bareos-webui/
```

Login fields:
- **Username** — the `Name` of the Console resource you created (e.g., `admin`)
- **Password** — the `Password` from that Console resource
- **Director** — the `[section]` name from `directors.ini` (e.g., `local-director`)

After login you land on the **Dashboard** showing:
- Running jobs
- Last 24 h job summary (OK / Warning / Error counts)
- Last used volumes

#### Trigger a manual job run from the WebUI

1. Navigate to **Jobs → Run**
2. Select a Job, Level, Client, FileSet, Pool
3. Click **Run**
4. Monitor progress on **Jobs → Running**

#### Perform a file-tree restore from the WebUI

1. Navigate to **Restore**
2. Select **Client** → **Backup Job** → set merge options
3. Browse the file tree; tick individual files/directories
4. Set **Restore to client**, **Replace files**, **Restore location** (`/` for in-place, `/tmp/restore/` for staging)
5. Click **Restore**
6. Monitor at **Jobs → Running**

[↑ Table of Contents](#table-of-contents)

---

## Lab Exercises

### Lab 07-1: Complete Bareos installation

```bash
# Follow Sections 2.1–2.7:
# 1. Add Bareos repo
# 2. Install all packages
# 3. Configure MariaDB
# 4. Create catalog database
# 5. Configure firewall
# 6. Start all services

# Verify all services are running
sudo systemctl status bareos-dir bareos-sd bareos-fd mariadb

# Connect with bconsole
sudo bconsole
*status dir
*status storage=FileStorage
*status client=bareos-fd
```

### Lab 07-2: Configure a FileSet and Job

```bash
# 1. Create the DefaultFileSet (Section 4.6)
# 2. Create a job to back up /etc and /home (Section 4.10)
# 3. Validate config
sudo bareos-dir -t

# 4. Reload Director
sudo systemctl reload bareos-dir

# 5. Verify in bconsole
sudo bconsole
*show filesets
*show jobs
```

### Lab 07-3: Run a backup and verify

```bash
sudo bconsole

# Run a full backup immediately
*run job=BackupLocalServer level=Full yes

# Watch the job run
*messages
*status dir

# After completion: list the job
*list jobs last=5

# List the volume created
*list volumes

# List files backed up
*list files jobid=1
```

### Lab 07-4: Restore a single file

```bash
sudo bconsole
*restore

# Select: 5 (most recent backup for a client)
# Select: bareos-fd
# Select: DefaultFileSet
# Navigate: cd etc; add hostname; done
# Set Where: /restore/
# Confirm: yes

# After restore completes:
cat /restore/etc/hostname
```

### Lab 07-5: Add a second client

```bash
# On backup-client-1:
# Install bareos-filedaemon
# Configure myself.conf and director resource
# Start bareos-fd

# On backup-server:
# Add client resource
# Add job for client
# Reload Director
# Test: status client=backup-client-1

# Run first backup for client
sudo bconsole
*run job=BackupClient1 level=Full yes
```

### Lab 07-6: Configure TLS

```bash
# Follow Section 14 to generate certificates and enable TLS
# Restart all daemons and verify in bconsole status
sudo bconsole
*status dir
*status storage=FileStorage
*status client=bareos-fd
# All should show TLS: enabled
```

### Lab 07-7: Test bextract (bare-metal restore simulation)

```bash
# Without a running Director, extract from volume
sudo bls -V Full-0001 /backup/bareos 2>/dev/null | head -20
sudo mkdir -p /restore/bextract
sudo bextract \
  -V Full-0001 \
  /backup/bareos \
  /restore/bextract/

ls /restore/bextract/etc/
```

### Lab 07-8: Configure email alerting

```bash
# 1. Edit Messages resource to add email
# 2. Install and configure postfix or bsmtp
# 3. Run a job and verify email is received
```

### Lab 07-9: Install and use Bareos WebUI

**Step 1:** Install the WebUI stack.

```bash
[server]# dnf install -y bareos-webui httpd php-fpm mod_fcgid
systemctl enable --now php-fpm httpd
```

**Step 2:** Allow Apache to reach the Director.

```bash
[server]# setsebool -P httpd_can_network_connect on
getsebool httpd_can_network_connect
# Expected: httpd_can_network_connect --> on
```

**Step 3:** Open the firewall.

```bash
[server]# firewall-cmd --permanent --add-service=http
firewall-cmd --reload
```

**Step 4:** Set the PHP timezone to match your system.

```bash
[server]# timedatectl | grep "Time zone"
# e.g., Time zone: Europe/London
sed -i 's|^;date.timezone.*|date.timezone = Europe/London|' /etc/php.ini
systemctl restart php-fpm
```

**Step 5:** Create a full-admin WebUI user.

```bash
[server]# bconsole << 'EOF'
reload
configure add console name=admin password=WebUIP@ss1 profile=webui-admin tlsenable=false
reload
quit
EOF
```

**Step 6:** Verify the profile shipped by the package exists.

```bash
[server]# cat /etc/bareos/bareos-dir.d/profile/webui-admin.conf
# Should show the webui-admin Profile with CommandACL entries
```

**Step 7:** Reload the Director and restart Apache.

```bash
[server]# bareos-dir -t && systemctl reload bareos-dir
systemctl reload httpd
```

**Step 8:** Log in.

```
Open: http://192.168.100.10/bareos-webui/
Username: admin
Password: WebUIP@ss1
Director: local-director   (or whatever [section] name is in directors.ini)
```

**Step 9:** Trigger a job from the WebUI.

```
Jobs → Run → select BackupLocalServer → Level: Full → Run
```

Watch it complete on: **Jobs → Running** (auto-refreshes every 60 s by default).

**Step 10:** Create a read-only monitoring user and confirm restricted access.

```bash
[server]# cat > /etc/bareos/bareos-dir.d/profile/webui-readonly.conf << 'EOF'
Profile {
  Name = "webui-readonly"
  CommandACL = .api, .help, use, version, status
  CommandACL = list, llist
  CommandACL = .clients, .jobs, .filesets, .pools, .storages, .defaults, .schedule
  CommandACL = .bvfs_lsdirs, .bvfs_lsfiles, .bvfs_update
  CommandACL = .bvfs_get_jobids, .bvfs_versions, .bvfs_restore
  JobACL      = *all*
  ScheduleACL = *all*
  CatalogACL  = *all*
  PoolACL     = *all*
  StorageACL  = *all*
  ClientACL   = *all*
  FileSetACL  = *all*
  WhereACL    = *all*
}
EOF

bconsole << 'EOF'
configure add console name=monitor password=MonitorP@ss profile=webui-readonly tlsenable=false
reload
quit
EOF
```

Log in as `monitor` — confirm that **Jobs → Run** is absent and no destructive actions are available.

**Step 11:** Perform a file restore via the WebUI file-tree browser.

```
Restore →
  Client: bareos-fd
  Backup Jobs: (select the Full job from Step 9)
  Merge jobs: Yes
  → Browse the file tree
  → Navigate to /etc/ → tick hostname → tick hosts
  Restore to client: bareos-fd
  Replace files: always
  Restore location: /tmp/webui-restore/
  → Restore
```

Verify on the server:

```bash
[server]# ls -la /tmp/webui-restore/etc/
cat /tmp/webui-restore/etc/hostname
```

[↑ Table of Contents](#table-of-contents)

---

## Review Questions

1. What are the four main Bareos components and what does each do?
2. What is the Catalog and why is it NOT the backup data?
3. What is the difference between a JobDef and a Job?
4. Explain the Pool/Volume lifecycle in Bareos.
5. What do the `Full Backup Pool`, `Incremental Backup Pool`, and `Differential Backup Pool` directives in a Job do?
6. What two files must have matching passwords for the File Daemon to accept connections from the Director?
7. What does `Action On Purge = Truncate` do in a Pool resource?
8. What is a Bootstrap file and what is `bextract` used for?
9. What bconsole command estimates how many files and bytes a job would back up?
10. How do you prune expired File records from the catalog to save space?
11. What is the `FileSet Options` directive `One FS = yes` for?
12. What bconsole command lists all jobs that have failed (status=E)?
13. In the Schedule syntax, what does `Run = Full 1st sun at 01:05` mean?
14. Why must TLS certificates be signed by the same CA for all Bareos daemons?
15. What is the difference between `prune` (bconsole) and `purge volume`?
16. Why is `setsebool -P httpd_can_network_connect on` required when running the Bareos WebUI on RHEL 10?
17. What is the Bvfs cache, and what happens if it is not updated after large backup jobs?

[↑ Table of Contents](#table-of-contents)

---

## Answers to Review Questions

1. **Director (bareos-dir):** orchestrates all operations, reads config, schedules jobs, coordinates FD and SD. **Storage Daemon (bareos-sd):** manages physical storage, writes/reads data to disk or tape. **File Daemon (bareos-fd):** runs on clients, reads the filesystem, sends data to SD. **Catalog (MariaDB):** stores metadata about what was backed up — not the backup data itself.
2. The **Catalog** is a relational database (MariaDB) storing records of: which files were backed up, on which job, on which volume, when. It does NOT store the actual file data. The catalog is needed to efficiently find and restore files but can be rebuilt from volumes if lost.
3. A **JobDef** is a reusable template (default values). A **Job** inherits from a JobDef and overrides specific values. Multiple jobs can share the same JobDef, reducing config repetition.
4. **Volume lifecycle:** New → Append (data being written) → Full (volume full or MaxVolumeJobs reached) → Used (no more data being written) → Purged (all jobs in the volume have exceeded their retention) → Recycled (can be overwritten). `Auto Prune = yes` and `Recycle = yes` enable automatic lifecycle management.
5. These directives route each backup level to a different Pool. Full backups go to the Full Pool (longer retention), Incrementals go to the Incremental Pool (shorter retention), Differentials to their pool. This allows different retention and volume management per backup level.
6. The password in the **Director's Client resource** (`bareos-dir.d/client/hostname.conf`) must match the password in the **File Daemon's Director resource** (`bareos-fd.d/director/bareos-dir.conf`) on the client machine.
7. `Action On Purge = Truncate` instructs Bareos to zero-out (truncate to 0 bytes) the volume file on disk after it is purged, reclaiming the disk space immediately rather than waiting until the file is overwritten.
8. A **Bootstrap file** (`.bsr`) is a text file generated during restore planning that lists which volumes, sessions, and file records need to be read to perform a restore. `bextract` is a standalone tool that reads Bareos volumes directly without a running Director, using a BSR file or volume list — used for emergency bare-metal restores when the Director is unavailable.
9. `*estimate job=JOBNAME level=Full` — shows the number of files and estimated bytes without running the actual backup.
10. From bconsole: `*prune files client=CLIENTNAME yes` — removes expired File records from the catalog. This is the largest single action for reducing catalog size. The actual backup data on volumes is unaffected.
11. `One FS = yes` tells the File Daemon not to cross filesystem boundaries. A directory backed up with `One FS = yes` will not descend into subdirectories that are separate mounts (e.g., if `/var` is a separate LV, backing up `/` with `One FS = yes` will not include `/var`). This prevents double-backups and surprises.
12. `*list jobs status=E` — `E` = Error. Also `*list jobs status=W` for warnings, `*list jobs status=T` for terminated normally.
13. Run a **Full** backup on the **first Sunday** of each month at **01:05 AM**.
14. TLS mutual authentication requires that each daemon can verify the other's certificate was signed by a trusted Certificate Authority. Using the same CA means all daemons trust each other's certificates automatically. Using different CAs would require each daemon to trust all the other CAs, which is complex and error-prone.
15. `*prune` removes **catalog records** for jobs/volumes/files that have exceeded their retention period — it does not touch the actual volume files on disk. `*purge volume=NAME` marks all jobs in a specific volume as expired in the catalog regardless of retention settings, making the volume eligible for recycling. Purge is a manual override; prune is automatic.

16. On RHEL 10 with SELinux enforcing, the `httpd_t` domain (Apache) is not allowed to make outbound network connections by default. The WebUI must connect to the Director on TCP port 9101. Without the boolean, every login attempt is silently blocked by SELinux, and `ausearch -m avc` will show denied `connect` calls from `httpd_t` to port 9101. The `-P` flag makes the boolean persistent across reboots.

17. The **Bvfs (Bareos Virtual Filesystem) cache** is a pre-computed index of the directory and file metadata from the catalog, used by the WebUI restore module to render the interactive file-tree browser. If the cache is not updated after a large backup job, the WebUI must build it on demand when a user opens the restore tree — which can time out for jobs with millions of files. The fix is to add a `RunScript { Console = ".bvfs_update jobid=%i" RunsWhen = After RunsOnClient = No }` directive to each Job or JobDef, so the cache is refreshed immediately after every backup completes.

[↑ Table of Contents](#table-of-contents)

---

*Previous: [06 — Restic](06-restic.md)*
*Next: [08 — Amanda](08-amanda.md)*

© 2026 Jaco Steyn — Licensed under CC BY-SA 4.0
