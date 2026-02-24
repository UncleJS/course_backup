# Module 09 — Backup Strategy and Automation

## Learning Objectives

By the end of this module you will be able to:

- Design a complete backup strategy appropriate to your environment
- Choose and combine full, incremental, and differential backup types correctly
- Implement Grandfather-Father-Son (GFS) rotation and Tower of Hanoi rotation schemes
- Define retention policies based on RPO and RTO requirements
- Schedule backups with both `cron` and `systemd` timers (with feature comparison)
- Build failure alerting via email, syslog, and systemd journal
- Automate pre- and post-backup hooks (database freeze, LVM snapshot, health checks)
- Create a unified backup wrapper that coordinates multiple tools

---

## Table of Contents

- [1. Strategy Before Tools](#1-strategy-before-tools)
- [2. Backup Types — Refresh and Scheduling Implications](#2-backup-types--refresh-and-scheduling-implications)
  - [2.1 Full backup](#21-full-backup)
  - [2.2 Incremental backup](#22-incremental-backup)
  - [2.3 Differential backup](#23-differential-backup)
  - [2.4 Comparison table](#24-comparison-table)
- [3. Rotation Schemes](#3-rotation-schemes)
  - [3.1 Grandfather-Father-Son (GFS)](#31-grandfather-father-son-gfs)
  - [3.2 Tower of Hanoi](#32-tower-of-hanoi)
  - [3.3 Continuous Data Protection (CDP)](#33-continuous-data-protection-cdp)
- [4. Retention Policies](#4-retention-policies)
  - [4.1 Defining retention tiers](#41-defining-retention-tiers)
  - [4.2 Compliance drivers](#42-compliance-drivers)
  - [4.3 Restic retention policy (declarative example)](#43-restic-retention-policy-declarative-example)
  - [4.4 Bareos retention (in Pool resource — from Module 07)](#44-bareos-retention-in-pool-resource--from-module-07)
- [5. Scheduling: cron vs systemd Timers](#5-scheduling-cron-vs-systemd-timers)
  - [5.1 Feature comparison](#51-feature-comparison)
  - [5.2 cron scheduling syntax](#52-cron-scheduling-syntax)
  - [5.3 systemd timer units](#53-systemd-timer-units)
- [6. Pre- and Post-Backup Hooks](#6-pre--and-post-backup-hooks)
  - [6.1 Common pre-backup hooks](#61-common-pre-backup-hooks)
  - [6.2 Common post-backup hooks](#62-common-post-backup-hooks)
  - [6.3 Hook framework in a wrapper script](#63-hook-framework-in-a-wrapper-script)
- [7. Failure Alerting](#7-failure-alerting)
  - [7.1 Email alerts via `mailx` / `sendmail`](#71-email-alerts-via-mailx--sendmail)
  - [7.2 Syslog / journald logging](#72-syslog--journald-logging)
  - [7.3 Webhook alert (Slack / generic HTTP)](#73-webhook-alert-slack--generic-http)
  - [7.4 systemd OnFailure alert unit](#74-systemd-onfailure-alert-unit)
- [8. Unified Backup Wrapper Script](#8-unified-backup-wrapper-script)
  - [/usr/local/sbin/backup-master.sh](#usrlocalsbinbackup-mastersh)
- [9. Log Rotation for Backup Logs](#9-log-rotation-for-backup-logs)
- [10. Monitoring Backup Health — Quick Checks](#10-monitoring-backup-health--quick-checks)
- [Lab 09 — Build a Full Backup Automation Stack](#lab-09--build-a-full-backup-automation-stack)
  - [Prerequisites](#prerequisites)
  - [Lab Part A — Deploy systemd Timer Pair](#lab-part-a--deploy-systemd-timer-pair)
  - [Lab Part B — Test Manual Trigger](#lab-part-b--test-manual-trigger)
  - [Lab Part C — Simulate Failure and Alert](#lab-part-c--simulate-failure-and-alert)
  - [Lab Part D — Log Rotation](#lab-part-d--log-rotation)
  - [Lab Part E — GFS Retention Simulation](#lab-part-e--gfs-retention-simulation)
- [Review Questions](#review-questions)
- [Answers to Review Questions](#answers-to-review-questions)

---

## 1. Strategy Before Tools

Every tool covered in previous modules is a mechanism. A *strategy* is the policy that answers:

| Question | Example answer |
|----------|----------------|
| **What** to back up | `/etc`, `/var`, `/home`, databases, VMs |
| **How often** | Full weekly, incremental nightly |
| **How many copies** | 3 copies minimum |
| **Where stored** | Local NVMe + remote SFTP + cloud S3 |
| **How long retained** | Daily 7 days, weekly 4 weeks, monthly 12 months |
| **How to test** | Automated restore check monthly |
| **Who is responsible** | Named owner per system |
| **Alert on failure** | Email + Slack webhook within 5 minutes |

Writing these answers down *before* configuring tools prevents the most common failure mode: backups that run but are never tested and fail silently for months.

[↑ Table of Contents](#table-of-contents)

---

## 2. Backup Types — Refresh and Scheduling Implications

### 2.1 Full backup

- Captures everything in scope
- Largest storage requirement; longest run time
- Fastest restore (single pass)
- Run: weekly or monthly

### 2.2 Incremental backup

- Captures only data changed since the **last backup of any type**
- Smallest storage; fastest run time
- Slowest restore (requires full + every incremental in chain)
- Run: daily between fulls

### 2.3 Differential backup

- Captures only data changed since the **last full backup**
- Storage grows each day until the next full
- Faster restore than incremental (requires full + one differential)
- Run: daily between fulls — popular for smaller environments

### 2.4 Comparison table

| Type | Storage use | Backup speed | Restore speed | Chain dependency |
|------|-------------|--------------|---------------|-----------------|
| Full | High | Slow | Fast | None |
| Incremental | Low/day | Fast | Slow | Full + all incrementals |
| Differential | Medium/day | Medium | Medium | Full + latest diff |
| Synthetic full | Medium | Medium | Fast | Built from previous backups |

[↑ Table of Contents](#table-of-contents)

---

## 3. Rotation Schemes

### 3.1 Grandfather-Father-Son (GFS)

GFS is the most widely used rotation scheme. It organises backups into three tiers:

```
Grandfather  = monthly full backup  (kept 12 months)
Father       = weekly full backup   (kept 4 weeks)
Son          = daily backup         (kept 7 days)
```

#### Weekly GFS schedule example

```
Mon  Tue  Wed  Thu  Fri  Sat  Sun
Inc  Inc  Inc  Inc  Inc  Inc  FULL (Father)
                                   ↳ last Sunday of month = Grandfather
```

Storage formula for GFS:
```
Total = (full_size × 12)            # grandfathers
      + (full_size × 4)             # fathers (rolling)
      + (incremental_size × 7)      # sons
```

For a 100 GB dataset with 5% daily change:
```
= (100 GB × 12) + (100 GB × 4) + (5 GB × 7)
= 1200 GB + 400 GB + 35 GB
= 1635 GB total retention storage
```

### 3.2 Tower of Hanoi

Uses a mathematical rotation to maximise retention with minimal media. Useful for tape environments.

```
Tape label:  A   B   A   C   A   B   A   D
Day:         1   2   3   4   5   6   7   8
Retention:  1d  2d  1d  4d  1d  2d  1d  8d
```

- Tape A used every other day (2-day maximum retention)
- Tape B used every 4th day (4-day maximum retention)
- Tape C used every 8th day (8-day retention)
- Each tape doubles the retention of the previous

### 3.3 Continuous Data Protection (CDP)

- Captures every write to a storage system in real time
- Implemented by tools like LVM thin snapshots, ZFS, or enterprise SAN features
- Very low RPO (seconds to minutes)
- High storage cost; requires dedicated infrastructure

[↑ Table of Contents](#table-of-contents)

---

## 4. Retention Policies

Retention policy = how long each class of backup is kept before expiration.

### 4.1 Defining retention tiers

```
Daily   backups → keep  7 days
Weekly  backups → keep  4 weeks (28 days)
Monthly backups → keep 12 months (365 days)
Yearly  backups → keep  7 years (legal/compliance minimum for many sectors)
```

### 4.2 Compliance drivers

| Regulation | Minimum retention | Scope |
|------------|------------------|-------|
| PCI-DSS | 1 year audit logs | Payment card environments |
| HIPAA | 6 years | US healthcare records |
| GDPR | Deletion required on request | EU personal data |
| SOX | 7 years financial records | US public companies |
| ISO 27001 | Defined in ISMS policy | Any certified organisation |

### 4.3 Restic retention policy (declarative example)

```bash
restic forget \
  --keep-daily   7  \
  --keep-weekly  4  \
  --keep-monthly 12 \
  --keep-yearly  7  \
  --prune
```

### 4.4 Bareos retention (in Pool resource — from Module 07)

```
VolumeRetention = 30 days
AutoPrune = yes
```

[↑ Table of Contents](#table-of-contents)

---

## 5. Scheduling: cron vs systemd Timers

Both can schedule backup jobs. The right choice depends on your environment.

### 5.1 Feature comparison

| Feature | cron | systemd timer |
|---------|------|---------------|
| Simplicity | High | Medium |
| Logging | Redirected manually | journald (automatic) |
| Dependency tracking | None | `Requires=`, `After=` |
| Missed job handling | Lost | `Persistent=true` catches up |
| Resource limits | None native | `CPUQuota=`, `MemoryMax=` |
| On-boot catch-up | No | Yes with `Persistent=true` |
| Monitoring | `cron.d` log | `systemctl status`, `journalctl` |
| Random delay | No | `RandomizedDelaySec=` |
| Multiple schedules | Multiple crontab lines | Multiple timer units |

**Recommendation:** Use systemd timers for production backups on RHEL 10. Use cron only for legacy scripts or simple single-command schedules.

### 5.2 cron scheduling syntax

```
# ┌─────── minute (0-59)
# │ ┌───── hour (0-23)
# │ │ ┌─── day of month (1-31)
# │ │ │ ┌─ month (1-12)
# │ │ │ │ ┌ day of week (0-7, 0=Sun)
# │ │ │ │ │
  * * * * *  command

# Examples:
0 2 * * *          # Every day at 02:00
0 2 * * 0          # Every Sunday at 02:00
0 2 1 * *          # First of every month at 02:00
0 2 * * 1-5        # Weekdays at 02:00
*/15 * * * *       # Every 15 minutes
0 2 * * 0 [ $(date +\%d) -le 07 ] && run_monthly_full.sh   # First Sunday of month
```

#### System-wide cron drop-in (recommended over user crontab)

```bash
# Create /etc/cron.d/backup
cat > /etc/cron.d/backup << 'EOF'
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root

# Incremental backup — nightly at 02:00
0 2 * * 1-6  root  /usr/local/sbin/backup-incremental.sh >> /var/log/backup/incremental.log 2>&1

# Full backup — Sunday at 01:00
0 1 * * 0    root  /usr/local/sbin/backup-full.sh >> /var/log/backup/full.log 2>&1
EOF
chmod 644 /etc/cron.d/backup
```

### 5.3 systemd timer units

A systemd timer requires two unit files: a `.timer` and a `.service`.

#### /etc/systemd/system/backup-incremental.service

```ini
[Unit]
Description=Nightly incremental backup
After=network-online.target
Wants=network-online.target
# Prevent overlapping runs
After=backup-full.service

[Service]
Type=oneshot
User=root
ExecStart=/usr/local/sbin/backup-incremental.sh
StandardOutput=journal
StandardError=journal
SyslogIdentifier=backup-incremental
# Resource limits so backup doesn't starve other services
CPUQuota=50%
IOWeight=50
Nice=10
# Retry on failure
Restart=on-failure
RestartSec=300

[Install]
WantedBy=multi-user.target
```

#### /etc/systemd/system/backup-incremental.timer

```ini
[Unit]
Description=Run nightly incremental backup at 02:00
Requires=backup-incremental.service

[Timer]
# Run Mon–Sat at 02:00
OnCalendar=Mon..Sat 02:00:00
# Catch up missed runs (e.g., server was off)
Persistent=true
# Spread load: random delay up to 5 minutes
RandomizedDelaySec=5min
# Accuracy window
AccuracySec=1min

[Install]
WantedBy=timers.target
```

#### /etc/systemd/system/backup-full.service

```ini
[Unit]
Description=Weekly full backup
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
User=root
ExecStart=/usr/local/sbin/backup-full.sh
StandardOutput=journal
StandardError=journal
SyslogIdentifier=backup-full
CPUQuota=60%
IOWeight=40
Nice=10
Restart=on-failure
RestartSec=600

[Install]
WantedBy=multi-user.target
```

#### /etc/systemd/system/backup-full.timer

```ini
[Unit]
Description=Run weekly full backup Sunday at 01:00
Requires=backup-full.service

[Timer]
OnCalendar=Sun 01:00:00
Persistent=true
RandomizedDelaySec=10min
AccuracySec=1min

[Install]
WantedBy=timers.target
```

#### Enable and start the timers

```bash
systemctl daemon-reload
systemctl enable --now backup-incremental.timer
systemctl enable --now backup-full.timer

# Verify
systemctl list-timers --all | grep backup
```

[↑ Table of Contents](#table-of-contents)

---

## 6. Pre- and Post-Backup Hooks

Hooks ensure the backup captures consistent data and cleans up properly.

### 6.1 Common pre-backup hooks

```bash
pre_backup_hooks() {
    # 1. Flush and freeze MySQL/MariaDB
    mysql -u root -e "FLUSH TABLES WITH READ LOCK; FLUSH LOGS;"
    # Note: use InnoDB hot backup or Percona XtraBackup for zero-downtime

    # 2. Create LVM snapshot for consistency
    lvcreate --snapshot --size 5G --name data_snap /dev/backupvg/data

    # 3. Unfreeze the database (snapshot is now isolated)
    mysql -u root -e "UNLOCK TABLES;"

    # 4. Verify snapshot exists
    lvs /dev/backupvg/data_snap || { echo "Snapshot failed"; exit 1; }
}
```

### 6.2 Common post-backup hooks

```bash
post_backup_hooks() {
    # 1. Remove LVM snapshot
    lvremove -f /dev/backupvg/data_snap

    # 2. Verify backup integrity
    restic check --repo /mnt/backup/restic-repo

    # 3. Record backup metadata
    echo "$(date -Iseconds) backup_size=$(du -sh /mnt/backup | cut -f1)" \
        >> /var/log/backup/metrics.log

    # 4. Send completion notification
    send_alert "SUCCESS" "Backup completed at $(date)"
}
```

### 6.3 Hook framework in a wrapper script

```bash
run_with_hooks() {
    local job_name="$1"
    local backup_fn="$2"

    log "=== Starting $job_name ==="
    pre_backup_hooks   || { send_alert "FAIL" "Pre-hooks failed for $job_name"; exit 1; }
    $backup_fn         || { send_alert "FAIL" "Backup failed for $job_name";    exit 1; }
    post_backup_hooks  || { send_alert "WARN" "Post-hooks failed for $job_name"; }
    log "=== $job_name complete ==="
}
```

[↑ Table of Contents](#table-of-contents)

---

## 7. Failure Alerting

### 7.1 Email alerts via `mailx` / `sendmail`

```bash
# Install mailx (requires a configured MTA or relay)
dnf install -y mailx

send_email_alert() {
    local subject="$1"
    local body="$2"
    local recipient="${ALERT_EMAIL:-root@localhost}"

    echo "$body" | mailx -s "[BACKUP ALERT] $subject" "$recipient"
}
```

#### Configure a simple SMTP relay (postfix)

```bash
dnf install -y postfix
systemctl enable --now postfix

# Relay through an external SMTP server
postconf -e "relayhost = [smtp.example.com]:587"
postconf -e "smtp_sasl_auth_enable = yes"
postconf -e "smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd"
postconf -e "smtp_tls_security_level = encrypt"

echo "[smtp.example.com]:587  user@example.com:password" > /etc/postfix/sasl_passwd
postmap /etc/postfix/sasl_passwd
chmod 600 /etc/postfix/sasl_passwd
systemctl restart postfix
```

### 7.2 Syslog / journald logging

```bash
# Write to syslog from a shell script
logger -t backup -p local0.err "Backup FAILED: $job_name at $(date)"
logger -t backup -p local0.info "Backup OK: $job_name at $(date)"

# Read backup logs from journal
journalctl -t backup -n 50
journalctl -t backup --since "24 hours ago"
journalctl -u backup-incremental.service --since yesterday
```

#### /etc/rsyslog.d/backup.conf — capture backup logs to a file

```
:programname, isequal, "backup"  /var/log/backup/syslog.log
& stop
```

```bash
systemctl restart rsyslog
```

### 7.3 Webhook alert (Slack / generic HTTP)

```bash
send_webhook_alert() {
    local status="$1"
    local message="$2"
    local webhook_url="${SLACK_WEBHOOK_URL:-}"

    [[ -z "$webhook_url" ]] && return 0

    local color="good"
    [[ "$status" == "FAIL" ]] && color="danger"
    [[ "$status" == "WARN" ]] && color="warning"

    curl -s -X POST "$webhook_url" \
        -H 'Content-type: application/json' \
        --data "{
            \"attachments\": [{
                \"color\": \"$color\",
                \"title\": \"Backup $status — $(hostname)\",
                \"text\": \"$message\",
                \"footer\": \"$(date -Iseconds)\"
            }]
        }"
}
```

### 7.4 systemd OnFailure alert unit

Trigger an alert unit automatically when a backup service fails:

#### /etc/systemd/system/backup-alert@.service

```ini
[Unit]
Description=Send backup failure alert for %i

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/backup-alert.sh %i
```

Then in each backup `.service` file add:

```ini
[Unit]
OnFailure=backup-alert@%n.service
```

[↑ Table of Contents](#table-of-contents)

---

## 8. Unified Backup Wrapper Script

This script ties together all previous modules into a single configurable wrapper.

### /usr/local/sbin/backup-master.sh

```bash
#!/usr/bin/env bash
# =============================================================================
# backup-master.sh — Unified backup orchestration script
# Compatible with RHEL 10, Bun-less shell, SELinux enforcing
# =============================================================================
set -euo pipefail
IFS=$'\n\t'

# ---------------------------------------------------------------------------
# Configuration
# ---------------------------------------------------------------------------
BACKUP_ROOT="/mnt/backup"
LOG_DIR="/var/log/backup"
LOCK_FILE="/run/backup-master.lock"
ALERT_EMAIL="${ALERT_EMAIL:-root@localhost}"
SLACK_WEBHOOK_URL="${SLACK_WEBHOOK_URL:-}"
HOSTNAME_SHORT="$(hostname -s)"
TIMESTAMP="$(date +%Y%m%d_%H%M%S)"
BACKUP_TYPE="${1:-incremental}"   # full | incremental | differential

# Restic configuration
RESTIC_REPO="${RESTIC_REPO:-${BACKUP_ROOT}/restic}"
RESTIC_PASSWORD_FILE="${RESTIC_PASSWORD_FILE:-/etc/backup/restic-password}"

# Directories to back up
BACKUP_SOURCES=(
    /etc
    /home
    /var/lib
    /opt
    /root
)

EXCLUDES=(
    /var/lib/docker
    /var/lib/containers
    /proc
    /sys
    /dev
    /run
    /tmp
    /var/tmp
)

# ---------------------------------------------------------------------------
# Logging
# ---------------------------------------------------------------------------
mkdir -p "$LOG_DIR"
LOG_FILE="${LOG_DIR}/backup-${TIMESTAMP}.log"
exec > >(tee -a "$LOG_FILE") 2>&1

log()  { echo "[$(date '+%Y-%m-%d %H:%M:%S')] [INFO]  $*"; logger -t backup -p local0.info  "$*"; }
warn() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] [WARN]  $*"; logger -t backup -p local0.warn  "$*"; }
err()  { echo "[$(date '+%Y-%m-%d %H:%M:%S')] [ERROR] $*"; logger -t backup -p local0.err   "$*"; }

# ---------------------------------------------------------------------------
# Locking (prevent overlapping runs)
# ---------------------------------------------------------------------------
acquire_lock() {
    if [[ -f "$LOCK_FILE" ]]; then
        local pid
        pid=$(cat "$LOCK_FILE")
        if kill -0 "$pid" 2>/dev/null; then
            err "Another backup is running (PID $pid). Aborting."
            exit 1
        else
            warn "Stale lock file found. Removing."
            rm -f "$LOCK_FILE"
        fi
    fi
    echo $$ > "$LOCK_FILE"
    trap 'rm -f "$LOCK_FILE"' EXIT
}

# ---------------------------------------------------------------------------
# Alerting
# ---------------------------------------------------------------------------
send_alert() {
    local status="$1"
    local message="$2"

    # Email
    echo "$message" | mailx -s "[BACKUP ${status}] ${HOSTNAME_SHORT}" "$ALERT_EMAIL" 2>/dev/null || true

    # Slack webhook
    if [[ -n "$SLACK_WEBHOOK_URL" ]]; then
        local color="good"
        [[ "$status" == "FAIL" ]] && color="danger"
        [[ "$status" == "WARN" ]] && color="warning"
        curl -s -X POST "$SLACK_WEBHOOK_URL" \
            -H 'Content-type: application/json' \
            --data "{\"attachments\":[{\"color\":\"$color\",\"title\":\"Backup $status — $HOSTNAME_SHORT\",\"text\":\"$message\"}]}" \
            || true
    fi

    # Journal
    logger -t backup -p local0.err "[$status] $message"
}

# ---------------------------------------------------------------------------
# Pre-backup hooks
# ---------------------------------------------------------------------------
pre_hooks() {
    log "Running pre-backup hooks..."

    # Freeze MariaDB if running
    if systemctl is-active --quiet mariadb; then
        log "Flushing MariaDB tables..."
        mysql -u root -e "FLUSH TABLES WITH READ LOCK; FLUSH LOGS;" || warn "MariaDB flush failed"
    fi

    # Create LVM snapshot if backupvg/data exists
    if lvs /dev/backupvg/data &>/dev/null; then
        log "Creating LVM snapshot..."
        lvcreate --snapshot --size 5G --name data_snap /dev/backupvg/data \
            || { err "LVM snapshot creation failed"; return 1; }

        # Unfreeze DB immediately after snapshot
        if systemctl is-active --quiet mariadb; then
            mysql -u root -e "UNLOCK TABLES;" || warn "MariaDB unlock failed"
        fi

        # Mount snapshot read-only
        mkdir -p /mnt/backup_snap
        mount -o ro,nouuid /dev/backupvg/data_snap /mnt/backup_snap \
            || { err "Snapshot mount failed"; lvremove -f /dev/backupvg/data_snap; return 1; }
        log "Snapshot mounted at /mnt/backup_snap"
    else
        # No snapshot: just unlock if we locked
        if systemctl is-active --quiet mariadb; then
            mysql -u root -e "UNLOCK TABLES;" 2>/dev/null || true
        fi
    fi
}

# ---------------------------------------------------------------------------
# Post-backup hooks
# ---------------------------------------------------------------------------
post_hooks() {
    log "Running post-backup hooks..."

    # Unmount and remove snapshot
    if mountpoint -q /mnt/backup_snap 2>/dev/null; then
        umount /mnt/backup_snap
        lvremove -f /dev/backupvg/data_snap
        log "Snapshot removed"
    fi

    # Record metrics
    echo "$(date -Iseconds) host=${HOSTNAME_SHORT} type=${BACKUP_TYPE} log=${LOG_FILE}" \
        >> "${LOG_DIR}/metrics.log"
}

# ---------------------------------------------------------------------------
# Backup functions
# ---------------------------------------------------------------------------
run_restic_backup() {
    local tag="$BACKUP_TYPE"
    log "Starting restic $BACKUP_TYPE backup..."

    local exclude_args=()
    for excl in "${EXCLUDES[@]}"; do
        exclude_args+=("--exclude" "$excl")
    done

    restic \
        --repo "$RESTIC_REPO" \
        --password-file "$RESTIC_PASSWORD_FILE" \
        backup \
        --tag "$tag" \
        --tag "host:${HOSTNAME_SHORT}" \
        --one-file-system \
        "${exclude_args[@]}" \
        "${BACKUP_SOURCES[@]}"

    log "Restic backup complete."
}

apply_retention() {
    log "Applying retention policy..."
    restic \
        --repo "$RESTIC_REPO" \
        --password-file "$RESTIC_PASSWORD_FILE" \
        forget \
        --keep-daily   7  \
        --keep-weekly  4  \
        --keep-monthly 12 \
        --keep-yearly  7  \
        --prune
    log "Retention applied."
}

run_rsync_backup() {
    log "Starting rsync backup..."
    local dest="${BACKUP_ROOT}/rsync/${HOSTNAME_SHORT}"
    local latest="${dest}/latest"
    local snapshot="${dest}/$(date +%Y/%m/%d_%H%M%S)"

    mkdir -p "$snapshot"

    rsync -aAXH \
        --delete \
        --link-dest="${latest}" \
        $(printf -- '--exclude=%s ' "${EXCLUDES[@]}") \
        "${BACKUP_SOURCES[@]}" \
        "${snapshot}/"

    # Update latest symlink
    ln -snf "$snapshot" "$latest"
    log "rsync snapshot created at $snapshot"
}

# ---------------------------------------------------------------------------
# Main
# ---------------------------------------------------------------------------
main() {
    acquire_lock

    log "========================================="
    log "Backup started: type=${BACKUP_TYPE} host=${HOSTNAME_SHORT}"
    log "========================================="

    pre_hooks || { send_alert "FAIL" "Pre-hooks failed"; exit 1; }

    run_restic_backup \
        || { send_alert "FAIL" "Restic backup failed — see ${LOG_FILE}"; post_hooks; exit 1; }

    # Apply retention only on full backup day (Sunday)
    if [[ "$BACKUP_TYPE" == "full" ]] || [[ "$(date +%u)" == "7" ]]; then
        apply_retention || warn "Retention policy application failed"
    fi

    post_hooks || warn "Post-hooks had errors"

    log "========================================="
    log "Backup COMPLETE: type=${BACKUP_TYPE}"
    log "========================================="
    send_alert "OK" "Backup completed successfully. Log: ${LOG_FILE}"
}

main "$@"
```

```bash
chmod 700 /usr/local/sbin/backup-master.sh
mkdir -p /etc/backup /var/log/backup

# Store restic password securely
echo "your-strong-password-here" > /etc/backup/restic-password
chmod 600 /etc/backup/restic-password
```

[↑ Table of Contents](#table-of-contents)

---

## 9. Log Rotation for Backup Logs

```bash
cat > /etc/logrotate.d/backup << 'EOF'
/var/log/backup/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 0640 root root
}
EOF
```

[↑ Table of Contents](#table-of-contents)

---

## 10. Monitoring Backup Health — Quick Checks

```bash
# 1. When did the last backup run?
journalctl -t backup --since "25 hours ago" | grep -E "COMPLETE|FAIL"

# 2. How large is the backup repository?
restic --repo /mnt/backup/restic --password-file /etc/backup/restic-password stats

# 3. Are all timers firing on schedule?
systemctl list-timers --all | grep backup

# 4. Check for failures in the last 7 days
journalctl -u backup-full.service -u backup-incremental.service \
    --since "7 days ago" | grep -i "fail\|error\|exit code"

# 5. Verify last snapshot exists
restic --repo /mnt/backup/restic --password-file /etc/backup/restic-password \
    snapshots --last

# 6. Quick integrity check
restic --repo /mnt/backup/restic --password-file /etc/backup/restic-password \
    check --read-data-subset=5%
```

[↑ Table of Contents](#table-of-contents)

---

## Lab 09 — Build a Full Backup Automation Stack

### Prerequisites

- Modules 02–06 completed
- `/dev/sdb` partitioned with `backupvg` VG (from Module 05)
- Restic repo initialised (from Module 06)

[↑ Table of Contents](#table-of-contents)

---

### Lab Part A — Deploy systemd Timer Pair

**Step 1:** Create the service and timer unit files.

```bash
[server]# cat > /etc/systemd/system/backup-incremental.service << 'EOF'
[Unit]
Description=Nightly incremental backup
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
User=root
ExecStart=/usr/local/sbin/backup-master.sh incremental
StandardOutput=journal
StandardError=journal
SyslogIdentifier=backup-incremental
CPUQuota=50%
Nice=10
EOF

cat > /etc/systemd/system/backup-incremental.timer << 'EOF'
[Unit]
Description=Nightly incremental backup timer
Requires=backup-incremental.service

[Timer]
OnCalendar=Mon..Sat 02:00:00
Persistent=true
RandomizedDelaySec=5min

[Install]
WantedBy=timers.target
EOF

cat > /etc/systemd/system/backup-full.service << 'EOF'
[Unit]
Description=Weekly full backup
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
User=root
ExecStart=/usr/local/sbin/backup-master.sh full
StandardOutput=journal
StandardError=journal
SyslogIdentifier=backup-full
CPUQuota=60%
Nice=10
EOF

cat > /etc/systemd/system/backup-full.timer << 'EOF'
[Unit]
Description=Weekly full backup timer
Requires=backup-full.service

[Timer]
OnCalendar=Sun 01:00:00
Persistent=true
RandomizedDelaySec=10min

[Install]
WantedBy=timers.target
EOF
```

**Step 2:** Deploy the wrapper script and enable timers.

```bash
[server]# cp /path/to/backup-master.sh /usr/local/sbin/backup-master.sh
chmod 700 /usr/local/sbin/backup-master.sh

mkdir -p /etc/backup /var/log/backup
echo "lab-test-password-$(hostname)" > /etc/backup/restic-password
chmod 600 /etc/backup/restic-password

systemctl daemon-reload
systemctl enable --now backup-incremental.timer
systemctl enable --now backup-full.timer
```

**Step 3:** Verify timers are active.

```bash
[server]# systemctl list-timers --all | grep backup
```

Expected output:
```
Mon 2026-02-23 02:00:00 UTC  ...  backup-incremental.timer  backup-incremental.service
Sun 2026-03-01 01:00:00 UTC  ...  backup-full.timer         backup-full.service
```

[↑ Table of Contents](#table-of-contents)

---

### Lab Part B — Test Manual Trigger

```bash
[server]# restic init --repo /mnt/backup/restic --password-file /etc/backup/restic-password

# Trigger incremental backup manually (bypasses timer schedule)
systemctl start backup-incremental.service

# Watch the logs live
journalctl -u backup-incremental.service -f

# Confirm it completed
systemctl status backup-incremental.service
```

[↑ Table of Contents](#table-of-contents)

---

### Lab Part C — Simulate Failure and Alert

**Step 1:** Temporarily break the restic repo path to force a failure.

```bash
[server]# RESTIC_REPO=/mnt/nonexistent /usr/local/sbin/backup-master.sh incremental
```

**Step 2:** Verify the failure is logged.

```bash
[server]# journalctl -t backup --since "5 minutes ago"
grep -i "FAIL" /var/log/backup/*.log | tail -5
```

**Step 3:** Restore correct config and confirm next run succeeds.

```bash
[server]# systemctl start backup-incremental.service
systemctl status backup-incremental.service
```

[↑ Table of Contents](#table-of-contents)

---

### Lab Part D — Log Rotation

```bash
[server]# cp /etc/logrotate.d/backup /etc/logrotate.d/backup.bak   # if exists
cat > /etc/logrotate.d/backup << 'EOF'
/var/log/backup/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 0640 root root
}
EOF

# Test logrotate config
logrotate -d /etc/logrotate.d/backup
logrotate -f /etc/logrotate.d/backup
ls -lh /var/log/backup/
```

[↑ Table of Contents](#table-of-contents)

---

### Lab Part E — GFS Retention Simulation

```bash
[server]# # Run 10 quick "backups" to build snapshot history
for i in $(seq 1 10); do
    restic backup --repo /mnt/backup/restic \
        --password-file /etc/backup/restic-password \
        --tag "lab-gfs" \
        /etc
    sleep 2
done

# List all snapshots
restic --repo /mnt/backup/restic --password-file /etc/backup/restic-password \
    snapshots --tag lab-gfs

# Apply GFS retention
restic --repo /mnt/backup/restic --password-file /etc/backup/restic-password \
    forget \
    --tag lab-gfs \
    --keep-daily 3 \
    --keep-weekly 2 \
    --keep-monthly 1 \
    --prune

# Verify snapshots after pruning
restic --repo /mnt/backup/restic --password-file /etc/backup/restic-password \
    snapshots --tag lab-gfs
```

[↑ Table of Contents](#table-of-contents)

---

## Review Questions

1. What is the difference between incremental and differential backups?
2. In a GFS rotation scheme, what are the three tiers, and how long is each retained?
3. What systemd timer directive ensures a missed backup is run after the machine restarts?
4. Why should backup scripts acquire a lock file before running?
5. What is the purpose of the pre-backup hook that flushes database tables?
6. Name three channels through which backup failures should be reported.
7. How does `CPUQuota=50%` in a systemd service unit benefit a production server during backup?
8. What command shows all systemd timers and their next trigger time?

[↑ Table of Contents](#table-of-contents)

---

## Answers to Review Questions

1. **Incremental** backs up data changed since the *last backup of any type* (full or incremental). **Differential** backs up data changed since the *last full backup only*. Differentials are larger per run but restore faster (only two backups needed: full + differential).

2. **GFS tiers:**
   - *Son* (daily): retained 7 days
   - *Father* (weekly full): retained 4 weeks
   - *Grandfather* (monthly full): retained 12 months

3. `Persistent=true` — instructs systemd to remember the last trigger time; if a run was missed (server off, timer inactive), the job fires immediately on next boot or timer activation.

4. A lock file prevents two concurrent backup processes from writing to the same repository simultaneously, which can corrupt archives or restic indexes. The lock also detects and removes stale locks from crashed prior runs.

5. `FLUSH TABLES WITH READ LOCK` ensures all in-memory dirty pages are written to disk and no further writes are allowed while the LVM snapshot is being created. This guarantees a crash-consistent (or even application-consistent) state in the backup data.

6. Email (via `mailx`/`sendmail`), systemd journal / syslog (`logger`), and webhook notifications (Slack, PagerDuty, etc.).

7. `CPUQuota=50%` caps the backup service to 50% of one CPU core, preventing the backup process from monopolising CPU and degrading foreground application performance during backup windows.

8. `systemctl list-timers --all`
