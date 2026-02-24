# Module 00 — Backup Theory & Fundamentals

## Learning Objectives

By the end of this module you will be able to:

- Define RPO, RTO, and explain their business impact
- Distinguish between full, incremental, differential, and synthetic full backups
- Apply the 3-2-1 backup rule to any environment
- Describe a backup lifecycle from creation to expiry
- Identify the most common causes of data loss and how backups address each
- Understand backup verification and why a backup that is never tested is not a backup

---

## Table of Contents

- [1. Why Backups Exist](#1-why-backups-exist)
- [2. Core Terminology](#2-core-terminology)
  - [Recovery Point Objective (RPO)](#recovery-point-objective-rpo)
  - [Recovery Time Objective (RTO)](#recovery-time-objective-rto)
  - [The relationship between RPO, RTO, and cost](#the-relationship-between-rpo-rto-and-cost)
- [3. Backup Types](#3-backup-types)
  - [3.1 Full Backup](#31-full-backup)
  - [3.2 Incremental Backup](#32-incremental-backup)
  - [3.3 Differential Backup](#33-differential-backup)
  - [3.4 Synthetic Full Backup](#34-synthetic-full-backup)
  - [3.5 Mirror / Sync](#35-mirror--sync)
  - [Comparison Table](#comparison-table)
- [4. The 3-2-1 Backup Rule](#4-the-3-2-1-backup-rule)
  - [Why each number matters](#why-each-number-matters)
  - [Modern extension: 3-2-1-1-0](#modern-extension-3-2-1-1-0)
  - [Practical 3-2-1 on RHEL 10](#practical-3-2-1-on-rhel-10)
- [5. Backup Lifecycle](#5-backup-lifecycle)
  - [Stage descriptions](#stage-descriptions)
  - [Retention policies](#retention-policies)
- [6. What Makes a Good Backup](#6-what-makes-a-good-backup)
- [7. Backup Consistency](#7-backup-consistency)
  - [Types of consistency](#types-of-consistency)
  - [Achieving consistency on RHEL 10](#achieving-consistency-on-rhel-10)
- [8. Backup vs. Replication vs. High Availability](#8-backup-vs-replication-vs-high-availability)
- [9. Backup Validation](#9-backup-validation)
- [10. Planning Your Backup Strategy](#10-planning-your-backup-strategy)
  - [Requirements gathering checklist](#requirements-gathering-checklist)
  - [Decision tree for choosing backup type](#decision-tree-for-choosing-backup-type)
- [Lab Exercises](#lab-exercises)
  - [Lab 00-1: Define your RPO and RTO](#lab-00-1-define-your-rpo-and-rto)
  - [Lab 00-2: Design a 3-2-1 backup plan](#lab-00-2-design-a-3-2-1-backup-plan)
  - [Lab 00-3: Identify what is NOT a backup](#lab-00-3-identify-what-is-not-a-backup)
- [Review Questions](#review-questions)
- [Answers to Review Questions](#answers-to-review-questions)

---

## 1. Why Backups Exist

Data loss happens. The cause is rarely one dramatic event — it is more commonly:

| Cause | Frequency | Example |
|-------|-----------|---------|
| Human error | Very common | `rm -rf /var/data` by accident |
| Hardware failure | Common | Disk head crash, RAID controller failure |
| Software bugs | Common | Database corruption, bad upgrade |
| Ransomware / malware | Increasing | Encrypts all accessible data |
| Natural disaster | Rare but catastrophic | Fire, flood, lightning |
| Power / storage failure | Common | Incomplete write, filesystem corruption |
| Logical corruption | Common | Application writes bad data over weeks |

A backup is a **copy of data that can be used to restore the original** after a loss event. This sounds obvious, but the subtleties — *what* data, *how often*, *stored where*, *kept how long*, *verified how* — define the difference between a working backup strategy and false security.

---

## 2. Core Terminology

### Recovery Point Objective (RPO)

**RPO** is the maximum amount of data loss, measured in time, that a business or system can tolerate.

> "If our database is destroyed at 14:47, how old can the backup be and still be acceptable?"

- RPO = 24 hours → a nightly backup is sufficient
- RPO = 1 hour → backups must run every hour
- RPO = 0 → real-time replication is required (this is not a backup strategy alone)

RPO is driven by **business requirements**, not technical ones. A sysadmin's job is to build a system that meets the RPO the business defines.

### Recovery Time Objective (RTO)

**RTO** is the maximum acceptable time to restore service after a failure.

> "How long can we be down before it costs too much / violates an SLA?"

- RTO = 4 hours → you have 4 hours to restore from backup
- RTO = 15 minutes → you need fast restore, possibly pre-staged restores or hot standby
- RTO = 0 → no downtime tolerated (high availability, not backup alone)

### The relationship between RPO, RTO, and cost

```
Low RPO  +  Low RTO  =  High cost (frequent backups, fast storage, hot standby)
High RPO +  High RTO =  Low cost  (nightly tape, slow restore)
```

Always document your RPO and RTO before choosing tools or designing a backup schedule.

---

## 3. Backup Types

### 3.1 Full Backup

A complete copy of **all** selected data, regardless of whether it changed since the last backup.

| Pros | Cons |
|------|------|
| Simple to restore (one archive) | Large storage requirement |
| No dependency chain | Long backup window |
| Easy to verify | High network/IO load |

### 3.2 Incremental Backup

Backs up only data that changed **since the last backup of any type** (last full or last incremental).

```
Day 1: Full    (100 GB)
Day 2: Incr    (2 GB  — changes since Day 1 Full)
Day 3: Incr    (1.5 GB — changes since Day 2 Incr)
Day 4: Incr    (3 GB  — changes since Day 3 Incr)
```

To restore Day 4: you need Day 1 Full + Day 2 Incr + Day 3 Incr + Day 4 Incr.

| Pros | Cons |
|------|------|
| Small backups after the first full | Complex restore (chain of archives) |
| Fast backup window | Restore fails if any link in chain is corrupt |
| Low storage overhead | |

### 3.3 Differential Backup

Backs up only data that changed **since the last full backup**, regardless of how many differentials have run.

```
Day 1: Full        (100 GB)
Day 2: Differential (2 GB  — changes since Day 1 Full)
Day 3: Differential (3.5 GB — all changes since Day 1 Full)
Day 4: Differential (6 GB  — all changes since Day 1 Full)
```

To restore Day 4: you need Day 1 Full + Day 4 Differential only.

| Pros | Cons |
|------|------|
| Simpler restore than incremental | Grows larger each day until next full |
| Only 2 sets needed to restore | More storage than incremental |

### 3.4 Synthetic Full Backup

A full backup assembled from a real full + subsequent incrementals, **without reading the source data again**. The backup server constructs the synthetic full from existing backup data.

Used in enterprise tools (like Bareos) to reduce backup window while maintaining simple restore.

### 3.5 Mirror / Sync

An exact real-time (or near real-time) copy of data. If a file is deleted on the source, it is deleted from the mirror. **This is not a backup** unless combined with versioning or snapshots — a ransomware attack would replicate instantly.

### Comparison Table

| Type | Storage | Backup Speed | Restore Speed | Restore Complexity |
|------|---------|--------------|---------------|--------------------|
| Full | High | Slow | Fast | Simple |
| Incremental | Low | Fast | Slower | Complex (chain) |
| Differential | Medium | Medium | Fast | Simple (2 sets) |
| Synthetic Full | Medium | Fast | Fast | Simple |
| Mirror | High | Fast | Fast | Risky (no history) |

---

## 4. The 3-2-1 Backup Rule

The 3-2-1 rule is the most widely cited backup best practice:

```
3  —  Keep at least 3 copies of your data
2  —  Store on at least 2 different media types
1  —  Keep at least 1 copy offsite
```

### Why each number matters

**3 copies:** The original plus two backups. If you have 2 copies (original + 1 backup) and both fail simultaneously (common when stored on the same system), you lose everything.

**2 media types:** Don't store all backups on the same type of storage. A disk RAID protects against disk failure but not against fire, flood, controller bugs, or ransomware. Examples: local disk + tape, local disk + cloud object storage, SSD + external USB.

**1 offsite:** Local backups protect against hardware failure. Offsite protects against site-level events: fire, flood, theft. "Offsite" can mean a cloud provider, a co-location facility, or a USB disk taken home.

### Modern extension: 3-2-1-1-0

An updated rule for ransomware-resistant environments:

```
3  copies
2  media types
1  offsite
1  offline / air-gapped (cannot be reached by ransomware)
0  errors in backup verification
```

### Practical 3-2-1 on RHEL 10

| Copy | Location | Media | Tool |
|------|----------|-------|------|
| Copy 1 | Source machine | Local disk (LVM) | LVM snapshot |
| Copy 2 | Local backup server | NAS / dedicated disk | rsync / Bareos |
| Copy 3 | Cloud / offsite | S3-compatible object storage | Restic |

---

## 5. Backup Lifecycle

Every backup has a lifecycle. Understanding it helps design retention policies.

```
 Creation ──► Verification ──► Storage ──► Aging ──► Expiry ──► Deletion/Overwrite
```

### Stage descriptions

| Stage | Description |
|-------|-------------|
| **Creation** | The backup job runs; data is copied/archived |
| **Verification** | Checksum, test restore, or catalog check confirms integrity |
| **Storage** | Backup is retained per policy (e.g., 30 days, 6 months) |
| **Aging** | Backup approaches its retention limit |
| **Expiry** | Retention period ends; backup is eligible for removal |
| **Deletion** | Backup data is removed to free space for new backups |

### Retention policies

A retention policy answers: *How many backups do we keep, and for how long?*

Common schemes:

| Scheme | Description |
|--------|-------------|
| **Count-based** | Keep the last N backups (e.g., last 7 dailies) |
| **Time-based** | Keep all backups within a time window (e.g., last 30 days) |
| **GFS (Grandfather-Father-Son)** | Daily (Son), Weekly (Father), Monthly (Grandfather) |
| **Tower of Hanoi** | Mathematical rotation scheme minimising tapes/media |

**GFS example:**
- Keep daily backups for 7 days
- Keep weekly backups (Sunday) for 4 weeks
- Keep monthly backups (1st of month) for 12 months
- Keep yearly backups (1st January) for 3 years

---

## 6. What Makes a Good Backup

A backup is only as good as your ability to restore from it. The following properties define a high-quality backup:

| Property | Description |
|----------|-------------|
| **Completeness** | All required data is included |
| **Consistency** | Data is in a consistent state (no half-written files, no open transactions) |
| **Integrity** | Backup is not corrupted; checksums pass |
| **Recoverability** | You have actually tested restoring from it |
| **Security** | Backup is encrypted and access-controlled |
| **Timeliness** | Backup is recent enough to meet your RPO |
| **Offsite** | At least one copy exists outside the primary location |

> **Rule:** A backup that has never been tested is not a backup. It is an assumption.

---

## 7. Backup Consistency

Backing up files that are actively being written to can produce an inconsistent backup — a backup that cannot be restored to a clean state.

### Types of consistency

| Type | Description | Method |
|------|-------------|--------|
| **Crash-consistent** | State as if power was suddenly cut | LVM snapshot |
| **Application-consistent** | Application is quiesced before backup | Pre/post scripts, flush+freeze |
| **Database-consistent** | DB transactions are committed or rolled back | `mysqldump`, `pg_dump`, hot backup agents |

### Achieving consistency on RHEL 10

- **For databases:** Use `mysqldump`, `mariabackup`, or `pg_basebackup` before backing up data files
- **For filesystems:** Use LVM snapshots (covered in Module 05) or XFS freeze (`xfs_freeze`)
- **For running VMs:** Use QEMU guest agent with filesystem freeze

---

## 8. Backup vs. Replication vs. High Availability

These are often confused but serve different purposes:

| Mechanism | Protects Against | Does NOT Protect Against |
|-----------|-----------------|--------------------------|
| **Backup** | Data loss, deletion, corruption | Downtime (unless fast restore) |
| **Replication** | Hardware failure, site failure | Data corruption, deletion (replicates instantly) |
| **High Availability (HA)** | Downtime, single node failure | Data loss, corruption, logical errors |

**They are complementary, not interchangeable.** A production system needs all three for full protection.

---

## 9. Backup Validation

After every backup, you should verify it. Methods:

| Method | Description | Cost |
|--------|-------------|------|
| **Checksum verification** | Verify stored data matches source checksums | Low |
| **Catalog check** | Tool confirms all files were catalogued | Low |
| **Test restore** | Restore to a test location and verify contents | Medium |
| **Full DR drill** | Restore entire system to a new machine and boot it | High |

Minimum recommended: checksum verification on every backup, test restore monthly, DR drill annually.

---

## 10. Planning Your Backup Strategy

Before touching a single tool, answer these questions:

### Requirements gathering checklist

```
[ ] What data must be backed up? (databases, configs, home dirs, application data)
[ ] What is the RPO for each data set?
[ ] What is the RTO for each data set?
[ ] How much storage is available locally?
[ ] Is offsite/cloud storage available?
[ ] Are there compliance requirements? (GDPR, HIPAA, PCI-DSS)
[ ] Who is responsible for running and monitoring backups?
[ ] Who is responsible for performing restores?
[ ] How will backups be monitored and alerted on failure?
[ ] When will restore tests be conducted?
```

### Decision tree for choosing backup type

```
Need to recover individual files quickly?
  └─ YES → Use file-level backup (tar, rsync, Restic, Bareos)
  └─ NO  → Need to recover entire filesystem?
             └─ YES → Use dump/xfsdump or LVM snapshot
             └─ NO  → Need offsite encrypted backup?
                        └─ YES → Use Restic
                        └─ NO  → rsync to NAS is sufficient
```

---

## Lab Exercises

### Lab 00-1: Define your RPO and RTO

For a hypothetical RHEL 10 server running:
- A MariaDB database (financial records)
- A web application serving 500 users
- Shared file storage for documents

Fill in this worksheet:

```
System Component   | RPO      | RTO       | Justification
-------------------|----------|-----------|---------------
MariaDB database   | ________ | _________ | _______________
Web app files      | ________ | _________ | _______________
Document storage   | ________ | _________ | _______________
/etc config files  | ________ | _________ | _______________
```

### Lab 00-2: Design a 3-2-1 backup plan

Using the lab environment described in the README, map each copy of data to a location and tool:

```
Copy 1 (local):    Location: ____________  Tool: ____________
Copy 2 (local alt): Location: ____________  Tool: ____________
Copy 3 (offsite):  Location: ____________  Tool: ____________
```

### Lab 00-3: Identify what is NOT a backup

For each of the following, state whether it qualifies as a backup and why:

1. A RAID-1 mirror of `/var/data`
2. An rsync job that syncs files to a NAS every hour (with `--delete`)
3. Restic snapshots stored in an S3 bucket with 30-day retention
4. A VM snapshot taken before a system upgrade
5. A MariaDB replication slave

---

## Review Questions

1. What does RPO stand for and what does it measure?
2. What is the difference between an incremental and a differential backup?
3. State the 3-2-1 rule and explain why each number is important.
4. Why is a mirror or sync NOT considered a complete backup strategy?
5. What is backup consistency, and what is the difference between crash-consistent and application-consistent?
6. Name three causes of data loss that backups protect against.
7. Why is a backup that has never been tested unreliable?
8. What is RTO, and how does it differ from RPO?
9. What is a GFS retention scheme?
10. Name two mechanisms that complement backups but do not replace them.

---

## Answers to Review Questions

1. **RPO** = Recovery Point Objective. It measures the maximum acceptable amount of data loss expressed as time — how old the most recent backup can be when a failure occurs.
2. **Incremental** backs up changes since the *last backup of any type*. **Differential** backs up changes since the *last full backup*. Incrementals are smaller per run; differentials require only two sets to restore.
3. **3** copies of data, on **2** different media types, with **1** copy offsite. Three copies ensure redundancy; two media types prevent a single media failure wiping all copies; offsite protects against site-level disasters.
4. A mirror replicates all changes including deletions and corruption. There is no historical copy to revert to. If ransomware encrypts the source, the mirror is encrypted too.
5. **Backup consistency** means the backed-up data represents a valid, recoverable state. **Crash-consistent** = data as of the moment of copy (like a power cut — journals may need replaying). **Application-consistent** = the application was quiesced, buffers flushed, before the backup was taken.
6. Any three of: human error (accidental deletion), hardware failure, software bugs/corruption, ransomware, natural disaster, power failure, logical corruption.
7. The backup media or archive may be corrupt, the restore process may fail, or data may be incomplete. Without a test restore, you cannot know if the backup is actually usable.
8. **RTO** = Recovery Time Objective. It measures the maximum acceptable time to restore service after a failure. RPO measures *data* loss tolerance; RTO measures *downtime* tolerance.
9. **GFS (Grandfather-Father-Son)** is a rotation scheme keeping: daily backups (Son) for ~1 week, weekly backups (Father) for ~1 month, monthly backups (Grandfather) for ~1 year.
10. Any two of: replication, high availability clustering, RAID, database hot standby.

---

*Next Module: [01 — RHEL 10 Filesystem & What to Back Up](01-rhel10-filesystem.md)*
