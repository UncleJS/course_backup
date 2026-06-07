# Module 11 — Backup Security and Compliance
[![CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey)](./LICENSE.md)
[![RHEL 10](https://img.shields.io/badge/platform-RHEL%2010-red)](https://access.redhat.com/products/red-hat-enterprise-linux)
[![Backup](https://img.shields.io/badge/Backup-RHEL%2010-blue)](https://access.redhat.com/products/red-hat-enterprise-linux)

## Learning Objectives

By the end of this module you will be able to:

- Encrypt backups at rest using GPG (tar/rsync pipelines) and Restic's built-in AES-256
- Manage encryption keys securely using GPG key rings and offline key storage
- Harden backup storage with correct file permissions, SELinux contexts, and restricted SSH
- Design an offsite and cloud backup strategy that meets compliance requirements
- Implement audit logging for all backup and restore operations
- Apply data retention rules driven by regulatory requirements (GDPR, HIPAA, PCI-DSS)
- Scan backups for compliance and detect sensitive data before archiving
- Understand threat models specific to backup infrastructure

---

## Table of Contents

- [1. Backup Threat Model](#1-backup-threat-model)
- [2. GPG Encryption for tar Backups](#2-gpg-encryption-for-tar-backups)
  - [2.1 Key generation](#21-key-generation)
  - [2.2 Encrypt a tar archive with GPG](#22-encrypt-a-tar-archive-with-gpg)
  - [2.3 Decrypt and restore](#23-decrypt-and-restore)
  - [2.4 GPG-encrypted backup script](#24-gpg-encrypted-backup-script)
- [3. Restic's Built-in Encryption](#3-restics-built-in-encryption)
  - [3.1 How Restic encryption works](#31-how-restic-encryption-works)
  - [3.2 Key/password management](#32-keypassword-management)
  - [3.3 Verify encryption on disk](#33-verify-encryption-on-disk)
- [4. Securing rsync with Restricted SSH](#4-securing-rsync-with-restricted-ssh)
  - [4.1 Dedicated backup SSH key](#41-dedicated-backup-ssh-key)
  - [4.2 Restricted `authorized_keys` on backup server](#42-restricted-authorized_keys-on-backup-server)
  - [4.3 rrsync wrapper (more flexible restriction)](#43-rrsync-wrapper-more-flexible-restriction)
  - [4.4 SSH server hardening for backup account](#44-ssh-server-hardening-for-backup-account)
- [5. File System Permissions for Backup Storage](#5-file-system-permissions-for-backup-storage)
  - [5.1 Backup directory structure with restrictive permissions](#51-backup-directory-structure-with-restrictive-permissions)
  - [5.2 SELinux context for backup directories](#52-selinux-context-for-backup-directories)
  - [5.3 Immutable flag to prevent ransomware deletion](#53-immutable-flag-to-prevent-ransomware-deletion)
- [6. Audit Logging for Backup Operations](#6-audit-logging-for-backup-operations)
  - [6.1 auditd rules for backup paths](#61-auditd-rules-for-backup-paths)
  - [6.2 Query audit logs](#62-query-audit-logs)
  - [6.3 Centralised syslog forwarding](#63-centralised-syslog-forwarding)
- [7. Offsite and Cloud Strategy](#7-offsite-and-cloud-strategy)
  - [7.1 3-2-1 rule reminder](#71-3-2-1-rule-reminder)
  - [7.2 Restic + AWS S3 (encrypted offsite)](#72-restic--aws-s3-encrypted-offsite)
  - [7.3 Restic + self-hosted MinIO (LAN or DMZ)](#73-restic--self-hosted-minio-lan-or-dmz)
  - [7.4 rsync over SSH to offsite server](#74-rsync-over-ssh-to-offsite-server)
- [8. Sensitive Data and GDPR/Compliance Considerations](#8-sensitive-data-and-gdprcompliance-considerations)
  - [8.1 What the GDPR says about backups](#81-what-the-gdpr-says-about-backups)
  - [8.2 Practical GDPR backup controls](#82-practical-gdpr-backup-controls)
  - [8.3 Scanning for sensitive data before backup (optional)](#83-scanning-for-sensitive-data-before-backup-optional)
- [9. Key Escrow and Disaster Key Recovery](#9-key-escrow-and-disaster-key-recovery)
  - [9.1 Key storage locations](#91-key-storage-locations)
  - [9.2 GPG key escrow](#92-gpg-key-escrow)
  - [9.3 Restic key recovery test](#93-restic-key-recovery-test)
- [10. Network Encryption in Transit](#10-network-encryption-in-transit)
  - [10.1 Verify SSH host key fingerprint before first connection](#101-verify-ssh-host-key-fingerprint-before-first-connection)
  - [10.2 Enforce StrictHostKeyChecking](#102-enforce-stricthostkeychecking)
  - [10.3 TLS for Restic REST server backend](#103-tls-for-restic-rest-server-backend)
- [11. Compliance Hardening Checklist](#11-compliance-hardening-checklist)
- [Lab 11 — Backup Security Hardening](#lab-11--backup-security-hardening)
  - [Prerequisites](#prerequisites)
  - [Lab Part A — GPG Encrypt a tar Archive](#lab-part-a--gpg-encrypt-a-tar-archive)
  - [Lab Part B — Harden Restic Password Storage](#lab-part-b--harden-restic-password-storage)
  - [Lab Part C — auditd Rules for Backup Paths](#lab-part-c--auditd-rules-for-backup-paths)
  - [Lab Part D — Immutable Backup Directory](#lab-part-d--immutable-backup-directory)
  - [Lab Part E — Compliance Self-Assessment](#lab-part-e--compliance-self-assessment)
- [Review Questions](#review-questions)
- [Answers to Review Questions](#answers-to-review-questions)

---

## 1. Backup Threat Model

Before implementing controls, understand what you are protecting against:

| Threat | Impact | Control |
|--------|--------|---------|
| Backup server compromised | Attacker reads all backed-up data | Encrypt at rest; separate credentials |
| Backup media stolen | Data exposed if unencrypted | Full encryption of all media |
| Ransomware encrypts backups | Cannot restore; forced to pay | Immutable / append-only repo; offsite copy |
| Insider threat | Malicious admin deletes backups | Multi-party approval for delete; audit logs |
| Key loss | Encrypted backups permanently unreadable | Key escrow; offline key copy |
| Accidental deletion | Backup set removed before expiry | `--lock` / immutable flags; retention policy |
| Man-in-the-middle on transfer | Backup traffic intercepted | TLS/SSH for all transfers; verify fingerprints |
| Credential leakage | `.env` file or script contains passwords in clear text | Secrets manager; password files with `chmod 600` |

[↑ Table of Contents](#table-of-contents)

---

## 2. GPG Encryption for tar Backups

GPG (GNU Privacy Guard) provides strong asymmetric encryption for archive files.

### 2.1 Key generation

```bash
# Generate a dedicated backup GPG key (no expiry recommended for backup keys)
gpg --full-generate-key
# Choose: (9) ECC (sign and encrypt), Curve 25519
# Real name: Backup Key backup-server.example.com
# Email: backup@example.com
# Passphrase: use a strong passphrase

# List keys
gpg --list-keys backup@example.com

# Export public key (distribute to all machines that need to encrypt)
gpg --export --armor backup@example.com > /etc/backup/backup-pubkey.asc

# Export secret key (store OFFLINE and in key escrow — NEVER on backup server)
gpg --export-secret-keys --armor backup@example.com > /OFFLINE/backup-secretkey.asc
chmod 400 /OFFLINE/backup-secretkey.asc
```

### 2.2 Encrypt a tar archive with GPG

```bash
# Encrypt for a specific recipient key
tar -czp \
    --xattrs --xattrs-include='*' --selinux --acls \
    /etc /home /var/lib \
| gpg --encrypt \
      --recipient backup@example.com \
      --trust-model always \
      --output /backup/etc-$(date +%Y%m%d).tar.gz.gpg

# Verify the encrypted file is not readable without the key
file /backup/etc-$(date +%Y%m%d).tar.gz.gpg
# Output: OpenPGP Public Key Encrypted Session Key Packet ...
```

### 2.3 Decrypt and restore

```bash
# Decrypt (requires secret key; run on machine with access to private key)
gpg --decrypt /backup/etc-20260224.tar.gz.gpg \
| tar -xzp \
      --xattrs --xattrs-include='*' --selinux --acls \
      -C /restore/staging/

# Or: decrypt to file first, then restore
gpg --output /tmp/etc-20260224.tar.gz \
    --decrypt /backup/etc-20260224.tar.gz.gpg

tar -xzf /tmp/etc-20260224.tar.gz \
    --xattrs --xattrs-include='*' --selinux --acls \
    -C /restore/staging/

rm -f /tmp/etc-20260224.tar.gz
```

### 2.4 GPG-encrypted backup script

```bash
#!/usr/bin/env bash
# /usr/local/sbin/backup-gpg.sh
set -euo pipefail

BACKUP_DIR="/backup/gpg"
GPG_RECIPIENT="backup@example.com"
TIMESTAMP="$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_DIR"

backup_encrypt() {
    local label="$1"; shift
    local sources=("$@")
    local dest="${BACKUP_DIR}/${label}-${TIMESTAMP}.tar.gz.gpg"

    tar -czp \
        --xattrs --xattrs-include='*' --selinux --acls \
        --exclude=/proc --exclude=/sys --exclude=/dev --exclude=/run \
        "${sources[@]}" \
    | gpg --batch --encrypt \
          --recipient "$GPG_RECIPIENT" \
          --trust-model always \
          --output "$dest"

    chmod 640 "$dest"
    echo "Encrypted backup: $dest ($(du -sh "$dest" | cut -f1))"
}

backup_encrypt "etc"    /etc
backup_encrypt "home"   /home
backup_encrypt "varlib" /var/lib
```

[↑ Table of Contents](#table-of-contents)

---

## 3. Restic's Built-in Encryption

Restic encrypts all data before it leaves the client using AES-256-CTR with a MAC (Poly1305). This is the preferred encryption method when using Restic.

### 3.1 How Restic encryption works

- Every pack file (chunks of backed-up data) is encrypted with AES-256
- The master key is random; it is stored wrapped (encrypted) by a key derived from the repository password using `scrypt`
- Even the repository index and snapshot metadata are encrypted
- The backup server only ever sees ciphertext — it cannot read backed-up data
- Password can be changed without re-encrypting all data (`restic key` command)

### 3.2 Key/password management

```bash
# The password is stored in a file (not in the command line or environment)
echo "long-random-passphrase-here" > /etc/backup/restic-password
chmod 600 /etc/backup/restic-password
chown root:root /etc/backup/restic-password

# Verify nobody else can read it
ls -l /etc/backup/restic-password
# -rw------- 1 root root 34 Feb 24 10:00 /etc/backup/restic-password

# List keys in repository
restic --repo /backup/restic --password-file /etc/backup/restic-password key list

# Add a second key (e.g., recovery key)
restic --repo /backup/restic --password-file /etc/backup/restic-password key add
# Enter new password when prompted (store this in your key escrow)

# Change the primary password
restic --repo /backup/restic --password-file /etc/backup/restic-password \
    key passwd

# Remove old key after password change
restic --repo /backup/restic --password-file /etc/backup/restic-password \
    key remove <key-id>
```

### 3.3 Verify encryption on disk

```bash
# The repository data directory contains only encrypted pack files
ls -lh /backup/restic/data/
# Output: directories with hex names containing .pack files

# Attempting to read a pack file directly shows ciphertext
xxd /backup/restic/data/ab/abcd1234... | head
# Output: binary garbage — encrypted content
```

[↑ Table of Contents](#table-of-contents)

---

## 4. Securing rsync with Restricted SSH

rsync over SSH provides transport encryption. Harden the SSH configuration for backup-only access.

### 4.1 Dedicated backup SSH key

```bash
# On backup client: generate a dedicated key for backup use only
ssh-keygen -t ed25519 -f /root/.ssh/backup_ed25519 -N "" \
    -C "backup-client@$(hostname)-$(date +%Y%m%d)"

chmod 600 /root/.ssh/backup_ed25519
chmod 644 /root/.ssh/backup_ed25519.pub
```

### 4.2 Restricted `authorized_keys` on backup server

```bash
# On backup server: add the client key with command restriction
# This key can ONLY run rsync — nothing else
cat >> /home/backupuser/.ssh/authorized_keys << 'EOF'
command="rsync --server --sender -vlogDtpre.iLsfxCIvu . /backup/clients/backup-client/",no-agent-forwarding,no-port-forwarding,no-pty,no-user-rc,no-X11-forwarding ssh-ed25519 AAAA...base64key... backup-client@backup-client-20260224
EOF

chmod 600 /home/backupuser/.ssh/authorized_keys
```

### 4.3 rrsync wrapper (more flexible restriction)

```bash
# Install rrsync (restricts rsync to a specific directory)
cp /usr/share/doc/rsync/support/rrsync /usr/local/bin/rrsync
chmod 755 /usr/local/bin/rrsync

# In authorized_keys — restrict to specific directory
command="/usr/local/bin/rrsync -ro /backup/clients/backup-client/",no-agent-forwarding,no-port-forwarding,no-pty ssh-ed25519 AAAA...
# -ro = read-only access (for pull backups from server to client)
# Omit -ro for read-write (push backups from client to server)
```

### 4.4 SSH server hardening for backup account

```bash
# /etc/ssh/sshd_config.d/backup-restrict.conf
cat > /etc/ssh/sshd_config.d/backup-restrict.conf << 'EOF'
Match User backupuser
    AllowTcpForwarding no
    X11Forwarding no
    PermitTTY no
    ForceCommand /usr/local/bin/rrsync /backup/clients/
    PasswordAuthentication no
    AuthorizedKeysFile /home/backupuser/.ssh/authorized_keys
EOF

sshd -t && systemctl reload sshd
```

[↑ Table of Contents](#table-of-contents)

---

## 5. File System Permissions for Backup Storage

### 5.1 Backup directory structure with restrictive permissions

```bash
# Create backup directory hierarchy
mkdir -p /backup/{restic,gpg,rsync,xfsdump}
chown -R root:root /backup
chmod 700 /backup

# Restic repo (only root can read/write)
chmod 700 /backup/restic

# GPG archives (root write, backup group read)
groupadd backupreaders 2>/dev/null || true
chmod 750 /backup/gpg
chgrp backupreaders /backup/gpg

# Verify
ls -la /backup/
```

### 5.2 SELinux context for backup directories

```bash
# Check default context for /mnt
ls -dZ /backup/

# Set an appropriate SELinux context. NOTE: there is no generic 'backup_t'
# type in the RHEL targeted policy — use an existing type (var_t shown here),
# or create a custom type via a policy module for real isolation
semanage fcontext -a -t var_t "/backup(/.*)?"
restorecon -Rv /backup/

# Verify
ls -dZ /backup/restic/

# If using NFS for backup storage, use the NFS context
semanage fcontext -a -t nfs_t "/mnt/nfs_backup(/.*)?"
```

### 5.3 Immutable flag to prevent ransomware deletion

```bash
# Make a SEALED ARCHIVE COPY immutable — never the live backup root, or the
# next backup run fails the moment it tries to write
chattr +i /backup/archive-2026Q1/
lsattr -d /backup/archive-2026Q1/

# Append-only on a log-style directory (new files OK, existing protected)
chattr +a /backup/logs/
lsattr -d /backup/logs/

# ⚠️ Do NOT chattr +a a restic repo's data/ directory — 'restic prune' must
# delete and rewrite pack files there. For restic immutability use
# rest-server --append-only or S3/MinIO object locking instead (Section 7).

# Remove immutable flag (requires root)
chattr -i /backup/archive-2026Q1/
```

[↑ Table of Contents](#table-of-contents)

---

## 6. Audit Logging for Backup Operations

All backup and restore operations must be logged with enough detail to reconstruct who did what and when.

### 6.1 auditd rules for backup paths

```bash
# /etc/audit/rules.d/backup.rules
cat > /etc/audit/rules.d/backup.rules << 'EOF'
# Log all access to backup password files
-w /etc/backup/ -p rwxa -k backup_secrets

# Log all writes and deletions in backup storage
-w /backup/ -p wa -k backup_storage

# Log execution of backup scripts
-w /usr/local/sbin/backup-master.sh -p x -k backup_exec
-w /usr/local/sbin/backup-gpg.sh    -p x -k backup_exec
-w /usr/local/sbin/verify-backup.sh -p x -k backup_exec

# Log restic binary execution — watch BOTH install paths (dnf/EPEL installs
# to /usr/bin; the manual install from Module 06 §2.2 uses /usr/local/bin)
-w /usr/bin/restic -p x -k backup_exec
-w /usr/local/bin/restic -p x -k backup_exec

# Log any attempt to remove backup archives
-a always,exit -F arch=b64 -S unlinkat -S rename -S renameat \
    -F dir=/backup -F success=1 -k backup_delete
EOF

augenrules --load
systemctl restart auditd
```

### 6.2 Query audit logs

```bash
# All backup-related events in the last 24 hours
ausearch -k backup_exec --start today | aureport -f -i

# All deletes in backup storage
ausearch -k backup_delete | head -50

# Who accessed backup secrets
ausearch -k backup_secrets | aureport --summary

# Export to human-readable report
aureport --start today --end now --file > /var/log/audit/backup-daily-report.txt
```

### 6.3 Centralised syslog forwarding

Forward backup audit logs to a remote, write-protected syslog server:

```bash
# /etc/rsyslog.d/remote-audit.conf
cat > /etc/rsyslog.d/remote-audit.conf << 'EOF'
# Forward all local0 messages (backup logs) to remote syslog over TLS
*.* action(type="omfwd"
    target="syslog.example.com"
    port="6514"
    protocol="tcp"
    action.resumeRetryCount="-1"
    StreamDriver="gtls"
    StreamDriverMode="1"
    StreamDriverAuthMode="x509/name"
    StreamDriverPermittedPeers="syslog.example.com")
EOF

systemctl restart rsyslog
```

[↑ Table of Contents](#table-of-contents)

---

## 7. Offsite and Cloud Strategy

The 3-2-1 rule requires at least one copy offsite. Here is how to implement this on RHEL 10.

### 7.1 3-2-1 rule reminder

```
3 copies of data
2 different storage media/types
1 offsite (geographically separate or cloud)
```

Extended to 3-2-1-1-0:
```
3 copies
2 media types
1 offsite
1 offline (air-gapped, never connected to the network)
0 errors after testing
```

### 7.2 Restic + AWS S3 (encrypted offsite)

```bash
# Configure AWS credentials
mkdir -p ~/.aws
cat > ~/.aws/credentials << 'EOF'
[backup]
aws_access_key_id     = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
EOF
chmod 600 ~/.aws/credentials

# Create S3-backed restic repository (data is encrypted before upload)
export AWS_PROFILE=backup
restic -r s3:s3.amazonaws.com/my-backup-bucket/restic-prod \
    --password-file /etc/backup/restic-password \
    init

# Back up to S3
restic -r s3:s3.amazonaws.com/my-backup-bucket/restic-prod \
    --password-file /etc/backup/restic-password \
    backup /etc /home /var/lib \
    --exclude /var/lib/docker

# Add S3 bucket policy to prevent deletion (object lock)
aws s3api put-bucket-versioning \
    --bucket my-backup-bucket \
    --versioning-configuration Status=Enabled \
    --profile backup

aws s3api put-object-lock-configuration \
    --bucket my-backup-bucket \
    --object-lock-configuration '{
        "ObjectLockEnabled": "Enabled",
        "Rule": {
            "DefaultRetention": {
                "Mode": "GOVERNANCE",
                "Days": 30
            }
        }
    }' \
    --profile backup
```

### 7.3 Restic + self-hosted MinIO (LAN or DMZ)

MinIO was covered in full in Module 06. Key security additions:

```bash
# Create a bucket with object locking enabled (immutable backups)
mc mb --with-lock myminio/backup-immutable

# Set retention (30 days). GOVERNANCE mode can be bypassed by any identity
# holding s3:BypassGovernanceRetention — for tamper-proof, ransomware-grade
# immutability use COMPLIANCE mode (nobody can shorten it, not even root):
mc retention set --default COMPLIANCE 30d myminio/backup-immutable

# Create a dedicated MinIO policy for the backup user (write-only, no delete)
cat > /tmp/backup-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:GetObject", "s3:ListBucket", "s3:GetBucketLocation"],
      "Resource": ["arn:aws:s3:::backup-immutable", "arn:aws:s3:::backup-immutable/*"]
    }
  ]
}
EOF
mc admin policy create myminio backup-write /tmp/backup-policy.json
mc admin user add myminio backupwriter $(openssl rand -base64 24)
mc admin policy attach myminio backup-write --user backupwriter
```

### 7.4 rsync over SSH to offsite server

```bash
# Nightly off-site push of local backup to remote site
rsync -aAXH --delete \
    -e "ssh -i /root/.ssh/backup_ed25519 -p 22022 -o StrictHostKeyChecking=yes" \
    /backup/restic/ \
    backupuser@offsite.example.com:/remote-backup/restic/

# Verify remote copy is consistent
ssh -i /root/.ssh/backup_ed25519 backupuser@offsite.example.com \
    "restic --repo /remote-backup/restic --password-file /etc/backup/restic-password check"
```

[↑ Table of Contents](#table-of-contents)

---

## 8. Sensitive Data and GDPR/Compliance Considerations

### 8.1 What the GDPR says about backups

- **Article 17** (Right to erasure): You must be able to delete personal data on request, including from backups.
- **Article 25** (Data minimisation): Do not back up more personal data than necessary.
- **Article 32** (Security): Must implement appropriate technical measures including encryption.
- **Article 33** (Breach notification): If backup media is lost or stolen, this may be a reportable breach — encryption mitigates this.

### 8.2 Practical GDPR backup controls

```bash
# 1. Document what personal data is in each backup scope
# Maintain a data register mapping: backup_set → data categories → retention

# 2. Use separate backup scopes for PII
# Don't mix /var/lib/mysql (PII) and /etc (config) in the same unencrypted archive

# 3. Track retention per backup set; delete expired backups on schedule
# Restic example: expire PII backups after 1 year
restic forget \
    --repo /backup/restic-pii \
    --password-file /etc/backup/restic-pii-password \
    --keep-yearly 1 \
    --prune

# 4. For right-to-erasure: document that encrypted backup sets containing
# the subject's data will expire on [date] and are not accessible without
# the decryption key. If the data is isolated (e.g., per-user backup),
# the relevant snapshot can be expired and pruned immediately.
```

### 8.3 Scanning for sensitive data before backup (optional)

```bash
# Install truffleHog or grep for patterns — example: find clear-text passwords
# This is a pre-backup scan, not a replacement for encryption.

# Find files containing private keys
grep -rl "BEGIN.*PRIVATE KEY" /etc/ 2>/dev/null

# Find files that might contain passwords in clear text (heuristic)
# (-P enables \s; -I skips binary files)
grep -rnPI "password\s*=\s*['\"]" /etc/ 2>/dev/null

# Find world-readable files in /etc that shouldn't be
find /etc -maxdepth 2 -perm /o+r -name "*.conf" -exec ls -la {} \;
```

[↑ Table of Contents](#table-of-contents)

---

## 9. Key Escrow and Disaster Key Recovery

Encryption is useless if you lose the key. Plan for key recovery before you need it.

### 9.1 Key storage locations

| Copy | Location | Access control |
|------|----------|----------------|
| Primary | `/etc/backup/restic-password` (chmod 600) | Root on backup server |
| Secondary | Password manager (Vault, Bitwarden, 1Password) | Named administrators |
| Tertiary | Printed, sealed envelope | Physical safe or safety deposit box |
| Recovery | Hardware security module (HSM) if available | Dual-control access |

### 9.2 GPG key escrow

```bash
# Export the secret key and encrypt it for multiple custodians
gpg --export-secret-keys --armor backup@example.com \
| gpg --encrypt \
      --recipient admin1@example.com \
      --recipient admin2@example.com \
      --output /ESCROW/backup-key-escrow.gpg

# Store escrow file in:
# - An offline USB drive
# - A separate encrypted volume
# - A physical safe

# Test recovery procedure annually:
gpg --decrypt /ESCROW/backup-key-escrow.gpg \
| gpg --import
gpg --list-secret-keys backup@example.com
```

### 9.3 Restic key recovery test

```bash
# Confirm the password file works (annual test in DR drill)
restic --repo /backup/restic \
    --password-file /etc/backup/restic-password \
    snapshots

# If password file is lost: use the secondary password from key escrow
echo "secondary-password-from-escrow" | \
restic --repo /backup/restic \
    --password-command "cat /dev/stdin" \
    snapshots
```

[↑ Table of Contents](#table-of-contents)

---

## 10. Network Encryption in Transit

### 10.1 Verify SSH host key fingerprint before first connection

```bash
# Get fingerprint of backup server before connecting
ssh-keyscan -t ed25519 192.168.100.10 | ssh-keygen -lf -
# Output: 256 SHA256:xxxx 192.168.100.10 (ED25519)

# Compare with known fingerprint from secure channel
# Then add to known_hosts:
ssh-keyscan -t ed25519 192.168.100.10 >> /root/.ssh/known_hosts
```

### 10.2 Enforce StrictHostKeyChecking

```bash
# /root/.ssh/config
cat >> /root/.ssh/config << 'EOF'
Host backup-server
    HostName 192.168.100.10
    User backupuser
    IdentityFile /root/.ssh/backup_ed25519
    StrictHostKeyChecking yes
    VerifyHostKeyDNS yes
    ServerAliveInterval 60
    ServerAliveCountMax 3
EOF
chmod 600 /root/.ssh/config
```

### 10.3 TLS for Restic REST server backend

If using `restic rest-server` as a backend:

```bash
# Run rest-server with TLS (auto-generates self-signed cert)
rest-server \
    --tls \
    --tls-cert /etc/pki/backup/server.crt \
    --tls-key  /etc/pki/backup/server.key \
    --path /backup/rest-server \
    --listen 0.0.0.0:8000 \
    --htpasswd-file /etc/backup/rest-users

# Client: trust the server certificate
export RESTIC_REPOSITORY="rest:https://192.168.100.10:8000/backup-client/"
export RESTIC_PASSWORD_FILE="/etc/backup/restic-password"
restic snapshots
```

[↑ Table of Contents](#table-of-contents)

---

## 11. Compliance Hardening Checklist

Use this checklist to assess and improve your backup security posture:

```
ENCRYPTION
[ ] All backup data encrypted at rest (AES-256 minimum)
[ ] All backup transfers encrypted in transit (SSH/TLS)
[ ] Encryption keys stored separately from backup data
[ ] Backup encryption keys held in escrow by >= 2 custodians
[ ] Password files have chmod 600 and are owned by root
[ ] Encryption strength verified annually (key algorithm still current)

ACCESS CONTROL
[ ] Backup system uses dedicated service account (not root where possible)
[ ] SSH keys are dedicated to backup (not shared with other services)
[ ] SSH access restricted by command= or ForceCommand
[ ] Backup storage directory has chmod 700 (or 750 for read group)
[ ] SELinux labels set correctly on all backup directories
[ ] No password authentication for SSH (key-only)

AUDIT AND MONITORING
[ ] All backup executions logged to syslog (local0 facility)
[ ] auditd rules watch backup password files and storage paths
[ ] Logs forwarded to remote syslog server (not only local)
[ ] Backup success/failure alerts sent to >= 2 channels
[ ] Log retention >= 1 year (PCI-DSS / SOX requirement)

RETENTION AND DELETION
[ ] Written retention policy exists per data category
[ ] Automated expiry implemented (restic forget --prune)
[ ] Expired backups are actually deleted (not just unfound)
[ ] GDPR data-erasure procedure documented
[ ] Immutable backup copies exist for ransomware protection

TESTING
[ ] Restore test performed after every configuration change
[ ] Monthly automated file restore verification
[ ] Quarterly full restore drill with RTO measurement
[ ] Annual bare-metal DR drill
[ ] Key recovery procedure tested annually
[ ] Offsite copy restore tested annually
```

[↑ Table of Contents](#table-of-contents)

---

## Lab 11 — Backup Security Hardening

### Prerequisites

- Module 06 (Restic) completed
- GPG installed: `dnf install -y gnupg2`
- Working restic repository at `/backup/restic`

[↑ Table of Contents](#table-of-contents)

---

### Lab Part A — GPG Encrypt a tar Archive

**Step 1:** Generate a lab GPG key.

```bash
[server]# gpg --batch --gen-key << 'EOF'
%no-protection
Key-Type: EDDSA
Key-Curve: ed25519
Subkey-Type: ECDH
Subkey-Curve: cv25519
Name-Real: Lab Backup Key
Name-Email: backup-lab@localhost
Expire-Date: 0
%commit
EOF

gpg --list-keys backup-lab@localhost
```

**Step 2:** Create an encrypted backup archive.

```bash
[server]# tar -czp --xattrs --xattrs-include='*' --selinux /etc/hostname /etc/hosts \
| gpg --batch --encrypt \
      --recipient backup-lab@localhost \
      --trust-model always \
      --output /backup/lab11-test.tar.gz.gpg

ls -lh /backup/lab11-test.tar.gz.gpg
file /backup/lab11-test.tar.gz.gpg
```

**Step 3:** Decrypt and restore.

```bash
[server]# gpg --decrypt /backup/lab11-test.tar.gz.gpg \
| tar -xzp --xattrs --xattrs-include='*' --selinux -C /restore/staging/
diff /etc/hostname /restore/staging/etc/hostname && echo "GPG restore: OK"
```

[↑ Table of Contents](#table-of-contents)

---

### Lab Part B — Harden Restic Password Storage

**Step 1:** Verify and fix password file permissions.

```bash
[server]# ls -l /etc/backup/restic-password
# Expected: -rw------- 1 root root
# Fix if wrong:
chown root:root /etc/backup/restic-password
chmod 600 /etc/backup/restic-password
```

**Step 2:** Add a recovery key to the restic repository.

```bash
[server]# restic --repo /backup/restic \
    --password-file /etc/backup/restic-password \
    key list

# Add a recovery key
echo "lab-recovery-password-$(openssl rand -hex 8)" > /etc/backup/restic-recovery-password
chmod 600 /etc/backup/restic-recovery-password

restic --repo /backup/restic \
    --password-file /etc/backup/restic-password \
    key add \
    --new-password-file /etc/backup/restic-recovery-password

# Confirm two keys exist
restic --repo /backup/restic \
    --password-file /etc/backup/restic-password \
    key list

# Test recovery key works
restic --repo /backup/restic \
    --password-file /etc/backup/restic-recovery-password \
    snapshots | head -5
echo "Recovery key: OK"
```

[↑ Table of Contents](#table-of-contents)

---

### Lab Part C — auditd Rules for Backup Paths

**Step 1:** Create audit rules.

```bash
[server]# cat > /etc/audit/rules.d/backup.rules << 'EOF'
-w /etc/backup/ -p rwxa -k backup_secrets
-w /backup/ -p wa -k backup_storage
-w /usr/local/sbin/backup-master.sh -p x -k backup_exec
-w /usr/bin/restic -p x -k backup_exec
-w /usr/local/bin/restic -p x -k backup_exec
EOF

augenrules --load
systemctl restart auditd
echo "Audit rules loaded"
```

**Step 2:** Trigger an audited event and check logs.

```bash
[server]# # Access the password file to trigger an audit event
cat /etc/backup/restic-password > /dev/null

# Check audit log
ausearch -k backup_secrets --start recent | tail -20
```

**Step 3:** Run a backup and check audit log for restic execution.

```bash
[server]# restic --repo /backup/restic \
    --password-file /etc/backup/restic-password \
    snapshots > /dev/null

ausearch -k backup_exec --start recent | tail -10
```

[↑ Table of Contents](#table-of-contents)

---

### Lab Part D — Immutable Backup Directory

```bash
[server]# # Create a separate "archive" directory that is append-only
mkdir -p /backup/archive
chattr +a /backup/archive
lsattr -d /backup/archive
# Output: -----a--------e------- /backup/archive

# Test: appending works (create new file)
touch /backup/archive/testfile && echo "Create: OK"

# Test: modifying existing file is denied
echo "test" >> /backup/archive/testfile \
    && echo "Append to file: OK (this is allowed with +a on directory)"

# Test: deleting existing file is denied
rm /backup/archive/testfile 2>&1 | grep -q "Operation not permitted" \
    && echo "Delete denied: OK"

# Cleanup for lab
chattr -a /backup/archive
rm -f /backup/archive/testfile
```

[↑ Table of Contents](#table-of-contents)

---

### Lab Part E — Compliance Self-Assessment

Run a quick self-assessment against the Section 11 checklist:

```bash
[server]# # Automated checks
PASS=0; FAIL=0
ok()   { echo "[PASS] $*"; ((PASS++)); }
fail() { echo "[FAIL] $*"; ((FAIL++)); }

# Encryption key permissions
[[ "$(stat -c '%a' /etc/backup/restic-password 2>/dev/null)" == "600" ]] \
    && ok "restic-password is chmod 600" || fail "restic-password permissions wrong"

# Backup directory permissions
[[ "$(stat -c '%a' /backup 2>/dev/null)" == "700" ]] \
    && ok "/backup is chmod 700" || fail "/backup permissions wrong"

# SELinux enforcing
sestatus | grep -q "enforcing" && ok "SELinux enforcing" || fail "SELinux not enforcing"

# auditd running
systemctl is-active --quiet auditd && ok "auditd running" || fail "auditd not running"

# Audit rules loaded
auditctl -l | grep -q "backup_secrets" && ok "backup audit rules loaded" || fail "backup audit rules missing"

# Restic repo is encrypted (check for key files)
[[ -d /backup/restic/keys ]] && ok "restic keys directory exists" || fail "restic repo not initialised"

echo ""
echo "Self-assessment: ${PASS} checks passed, ${FAIL} checks failed"
```

[↑ Table of Contents](#table-of-contents)

---

## Review Questions

1. What are the four main threats specific to backup infrastructure that encryption addresses?
2. How does Restic ensure the backup server cannot read backed-up data?
3. What is the purpose of the `command=` restriction in SSH `authorized_keys`?
4. What does `chattr +a` do to a directory, and why is it useful for backup storage?
5. Under GDPR Article 17, what must you do when a user requests erasure and their data is in an encrypted backup?
6. What is the 3-2-1-1-0 rule and what does each number represent?
7. Why should the GPG secret key never be stored on the backup server?
8. What `auditd` key name convention was used in the lab, and what benefit does naming keys provide?

[↑ Table of Contents](#table-of-contents)

---

## Answers to Review Questions

1. **Four backup-specific threats encryption addresses:**
   - Backup storage theft or physical access (attacker reads unencrypted archives)
   - Backup server compromise (attacker extracts data from the repository)
   - Cloud storage provider breach (provider or third-party reads your data)
   - Intercepted backup transfers (MITM reads data in transit)

2. Restic encrypts all pack files (data chunks), the index, and snapshot metadata on the *client* using a key derived from the repository password *before* any data is uploaded. The server receives and stores only ciphertext; it has no access to the decryption key.

3. The `command=` restriction in `authorized_keys` forces the SSH session to execute *only the specified command* regardless of what command the connecting client requests. This means a key configured with `command="rsync --server ..."` cannot be used to open a shell, run arbitrary commands, or access files outside the rsync scope.

4. `chattr +a` sets the *append-only* attribute on a directory. New files can be created inside it, but existing files cannot be deleted or overwritten. This protects backup archives against accidental deletion, malicious deletion, and ransomware that attempts to overwrite backups with encrypted versions.

5. Under GDPR, you must document that the backup set containing their data will be automatically expired and deleted on [date]. If the data is isolated (per-user snapshot), expire and prune it immediately. Encrypted backup data that has no associated accessible decryption key is generally considered effectively inaccessible and therefore compliant, but this must be documented in your DPIA (Data Protection Impact Assessment).

6. **3-2-1-1-0 extended rule:**
   - **3** — three total copies of data
   - **2** — two different media or storage types
   - **1** — one copy offsite (different geographic location or cloud)
   - **1** — one offline/air-gapped copy (not reachable over the network)
   - **0** — zero errors after verified restore testing

7. If the secret key is on the backup server and the server is compromised, the attacker can decrypt all past and future backups. The secret key must be on a separate, more tightly controlled system (e.g., an admin workstation, HSM, or offline media). The backup server needs only the *public* key to encrypt; it never needs the secret key.

8. The lab used keys named `backup_secrets`, `backup_storage`, and `backup_exec`. Naming audit rules with meaningful keys makes searching and reporting much easier: `ausearch -k backup_secrets` immediately filters to all events matching that key, without having to search by path or syscall across an unfiltered audit log.

[↑ Table of Contents](#table-of-contents)

---

*Previous: [10 — Restore Testing and Disaster Recovery Drills](10-restore-testing.md)*
*Next: [12 — Capstone Lab: Full Disaster Recovery Scenario](12-capstone-lab.md)*

© 2026 UncleJS — Licensed under CC BY-NC-SA 4.0
