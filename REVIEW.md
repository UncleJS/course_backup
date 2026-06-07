# Course Review — RHEL 10 Backup Procedure & Strategy

**Date:** 2026-06-07
**Scope:** all 13 modules + README — technical accuracy, pedagogy & flow, consistency & structure, currency (web/package-verified)
**Method:** per-module deep review (15 parallel review passes), cross-module consistency sweep, full link/anchor validation of all 15 markdown files, package-availability checks against live RHEL 10 repos, repository/version checks against download.bareos.org, EPEL, and GitHub releases, and empirical spot-tests of suspected shell bugs.

**Legend:** ✅ = fixed in this revision · 📋 = recommendation, not changed

---

## Executive summary

The course is well structured (consistent template, complete navigation chain, answer keys, good warning culture) and the majority of commands are accurate. However, the review found **a number of defects that would break labs or mislead students**, including an entire install path built on a removed feature (module 07's MariaDB catalog — Bareos has been PostgreSQL-only since v21), two packages taught as installable that don't exist in RHEL 10 repos (`dump`, `amanda`), several scripts with real shell bugs (verified by execution), one invented restic environment variable, a database-lock workflow that never actually holds the lock, and a disaster-simulation step that can wipe the wrong disk. 17 TOC anchors were broken. All clear-cut defects have been fixed in place; judgment calls are listed under *Recommendations*.

**Verified environment facts (ground truth, RHEL 10 host + web, June 2026):**

| Claim in course | Reality | Impact |
|---|---|---|
| `dnf install dump` works | **No `dump` package** in BaseOS/AppStream/EPEL 10 | M04 Part A unusable as a lab |
| Amanda installable from EPEL | **Not in EPEL 10** (EPEL 9 only, amanda-3.5.3) | M08 install fails on RHEL 10 |
| restic via EPEL | ✔ restic **0.18.1** in EPEL 10 | OK (course pinned 0.17.3 — outdated) |
| `dnf install mailx` | **Removed**; `s-nail` provides `mailx` | M09 alerting install fails |
| Bareos: "use EL_9 if EL_10 not available" | **EL_10 repo exists** at download.bareos.org | Stale instruction |
| Bareos catalog on MariaDB | **PostgreSQL only since Bareos 21** | M07 §2 cannot work |
| `dnf install fuse` (restic mount) | ✔ `fuse` 2.9.9 **does** exist in RHEL 10 | No change needed (suspected issue withdrawn) |

---

## Critical findings (broken commands / procedures)

### Module 02 — tar
- ✅ `02-tar.md` Lab 02-1: `echo "#!/bin/bash\necho hello" > …` wrote a literal `\n` (one malformed line, broken shebang). **Verified by execution.** Fixed with `printf`.
- ✅ SSH restore pipeline dropped `--selinux --numeric-owner` on the extract side, contradicting the module's own flag guidance. Fixed.

### Module 03 — rsync
- ✅ Production script: `SSH_OPTS="-e ssh -p … -i … -o …"` expanded **unquoted** — word-splitting made rsync consume `-p`/`-i` itself, silently discarding the SSH port and key on every remote run. Rebuilt as an array (`SSH_OPTS=(-e "ssh …")`); also quoted `"${EXCLUDES[@]}"`.
- ✅ `--bwlimit=2m` example had a comment **after** a line-continuation backslash — the command truncated mid-line and the source/dest line ran as a separate junk command. **Verified with `bash -x`.** Fixed.
- ✅ `mkdir -p /backup/clients/$(hostname -s from client)` — `hostname -s` takes no arguments; `from client` was a stray editing note. Fixed.
- ✅ Forced command `command="rsync --server --daemon ."` forces daemon-over-SSH mode and breaks the ordinary `rsync -e ssh` transfers the module teaches. Replaced with `rrsync` guidance + warning note.
- ✅ Trailing-slash "gotcha" example used different destinations so both variants produced the *same* result, defeating the lesson. Fixed to use one destination.

### Module 04 — dump/xfsdump
- ✅ xfsrestore flags table had `-r` = "resume" — **inverted**: `-r` is cumulative (incremental-chain) mode; resume is `-R`. Fixed table.
- ✅ §5.3 applied incremental restores **without `-r`**, so the documented chain-restore procedure does not work. Fixed (level-0 then increments, all with `-r`, plus `xfsrestorehousekeepingdir` cleanup); Answer 6 updated.
- ✅ `-e` documented and used as "estimate dump size" — actually means "allow excludes (via extended attributes)". xfsdump has **no estimate-only flag**. Fixed §4.4, §4.7 and RQ9 answer (use the `estimated dump size` line / `du -sx`).
- ✅ `dump -u` writes to `/etc/dumpdates`, not `/var/lib/dumpdates`. Fixed everywhere (heading, TOC anchor, examples, answer).
- ✅ Added RHEL 10 availability warning: `dump` is not installable (see ground-truth table); Part A is now flagged as reference/legacy; lab note updated.

### Module 05 — LVM snapshots
- ✅ MariaDB consistency workflow ran `FLUSH TABLES WITH READ LOCK` via one-shot `mysql -e` — the lock dies when that client exits, so the snapshot ran with **no lock at all**. Fixed to a hold-the-session workflow; Answer 9 corrected.
- ✅ PostgreSQL section used `pg_start_backup()`/`pg_stop_backup()` — **removed in PostgreSQL 15**; RHEL 10 ships 16+. Also the replacement API is session-scoped and cannot be split across `psql -c` calls. Fixed (single open session, `pg_backup_start/stop`, recommend `pg_basebackup`).
- ✅ Snapshot monitor piped through `grep -v ""` which matches everything inverted — the command printed **nothing**. Fixed (`awk 'NF>1'`).
- ✅ Plain `mount -o ro` of an XFS snapshot shown as a working step — it fails with a duplicate-UUID error while the origin is mounted. Fixed (nouuid mandatory, explained).

### Module 06 — restic
- ✅ §10.4 invented a `RESTIC_SFTP_ARGS` environment variable — **restic has no such variable**; the documented workflow silently ignored the SSH key. Replaced with the correct `~/.ssh/config` approach.
- ✅ Manual install piped `bunzip2 > /usr/local/bin/restic` without sudo (fails: root-owned dir); same for MinIO `wget -O /usr/local/bin/{minio,mc}`. Fixed.

### Module 07 — Bareos (systemic)
- ✅ The **entire catalog stack was taught as MariaDB/MySQL** — architecture diagram and table (port 3306), `bareos-database-mysql` package (doesn't exist for Bareos 21+), `mariadb-server` install + `mysql_secure_installation`, `mysql` verification commands, `DB Driver = mysql`, catalog-size queries, lab text, review answers. Current Bareos (the course installs from `current/`) is **PostgreSQL-only**. Converted everything to PostgreSQL (packages, `postgresql-setup --initdb`, peer-auth notes, `DB Driver = postgresql`/5432, `psql` verification, pg catalog-size query, labs, answers).
- ✅ Repo instructions pointed at `EL_9` "if EL_10 not yet available" — EL_10 is live (verified). Fixed both occurrences + hardcoded `EL_10` in the manual repo file (`$releasever` can expand to `10.0`).
- ✅ 15 broken TOC anchors (the slug for `bareos-dir.d/...` headings dropped the `d` of `.d`; one `lirectors` typo). All fixed — link checker now reports 0 broken anchors course-wide.
- ✅ `setsebool bareos_use_connection` — no such SELinux boolean exists (failure masked by `|| true`). Removed; SELinux section now points at the audit2allow workflow and says to verify available `bareos_*` types before `semanage fcontext`.
- ✅ `ClientACL = …, backup-client-fd` — name drift; the module defines the client as `backup-client-1` everywhere else (8 occurrences). Fixed.
- ✅ Device-resource comments: "Auto-mount device" attached to `Random Access`, and "Allow multiple volumes…" attached to `AutoChanger = no` (describes the opposite). Fixed both.
- ✅ `*restore client=bareos-fd where=/ all yes` restored over the live filesystem. Changed to a staging path + explicit warning.
- ✅ `bareos-dbcheck -B MyCatalog` — `-B` doesn't take a catalog operand; fixed to `-b`/interactive with `-c /etc/bareos`.
- ✅ `BackupCatalog` job referenced `Catalog` FileSet / `WeeklyCycleAfterBackup` schedule that are never defined in the module — annotated that these ship with the default install configs.

### Module 08 — Amanda
- ✅ Added prominent **EPEL 10 availability warning** (package does not exist; options listed). Removed bogus `amanda` metapackage and the invalid `amserverconfig --version || amanda --version` check.
- ✅ VTL scheme conflicted with chg-disk defaults: zero-padded `slot01…` dirs (chg-disk expects `slot1…`), a stray `data/` directory + `drive0` symlink, and a dead `chg-disk.conf` that nothing included while `amanda.conf` defined a second, different changer. Rebuilt: unpadded slots, `data -> slot1` symlink, single changer definition in `amanda.conf` with `num-slot`/`auto-create-slot` properties.
- ✅ `amadmin DailySet tape` was captioned "Label virtual tapes" — it only *shows* the next expected tape. Labeling section now uses `amlabel` (with unpadded `slot N`), `amadmin tape` demoted to verification.
- ✅ `amadmin force-level 1` → `force-level-1` (no such two-token subcommand).
- ✅ `amlabel DailySet -l DailySet-01` shown as "load the right VTL slot" — `amlabel` *writes labels* (destructive) and has no `-l`. Removed; restore now uses `amfetchdump`, which locates the volume via the changer.
- ✅ `amrestore -p DailySet DailySet-01` is not valid amrestore syntax (takes a device, not config+label). §13.2 rewritten around `amfetchdump -p <conf> <host> <disk>`.
- ✅ `amadmin find /etc/ssh/sshd_config` — `find` matches host/disk DLEs, not file paths. Fixed (`find backup-server /etc`, pointer to amrecover for single files).
- ✅ `amadmin delete` was described as "reset statistics" — it permanently removes the DLE and its history. Re-captioned with a warning.
- ✅ `amdump --no-taper` described as "verbose output" — it means dump-to-holding-disk-only. Fixed; `amadmin estimate` description fixed (shows stored estimates, not a dry run).
- ✅ amanda.conf: removed invalid global `ssh_keys`/`amandad_path` (they belong in dumptypes — `amandad_path` added to the `global` dumptype), removed duplicate `tapecycle`, removed global `maxdumps` with its wrong "maximum time" comment (correct count-semantics comment now in the dumptype), removed invalid `speed` tapetype parameter, fixed the stray "minimum size" comment over `dumpcycle`.
- ✅ amrecover section now warns that `amindexd`/`amidxtaped` services are required (no xinetd on RHEL 10) and how to run it server-local.
- ✅ §3.6 heading/text no longer implies xinetd exists on RHEL 10; TOC updated.
- ✅ Hostname drift `backup-client-1` → `backup-client` (the lab-env name) throughout the module; "AmandaWeb" → Zmanda Management Console; "no agent needed" claim corrected.

### Module 09 — strategy & automation
- ✅ `backup-master.sh` sets `IFS=$'\n\t'`, then relied on space-splitting of `$(printf -- '--exclude=%s ' …)` — with no space in IFS the excludes collapsed into **one mangled argument**. **Verified by execution.** Fixed with an exclude array (same pattern the script already used for restic).
- ✅ `OnFailure=backup-alert@%n.service` — `%n` includes `.service`, producing `backup-alert@backup-full.service.service`. Fixed to `%N` with an explanatory comment.
- ✅ `dnf install -y mailx` fails on RHEL 9+/10. Fixed to `s-nail` (provides the `mailx` command).
- ✅ Removed leaked template comment "Bun-less shell". Added a note that restic snapshots are inherently incremental — `BACKUP_TYPE` is organisational for restic, real for tar/xfsdump.

### Module 10 — restore testing
- ✅ `restic tag --add "restored-…"` without a snapshot ID tags **every** snapshot in the repo. Fixed (ID required, noted).
- ✅ Bare-metal bootloader rebuild was BIOS-only (`grub2-install /dev/sda` presented as universal). Added the UEFI path (RHEL: `dnf reinstall grub2-efi-x64 shim-x64`, config at `/boot/efi/EFI/redhat/grub.cfg`, `efibootmgr` for the NVRAM entry); Q7 answer updated.
- ✅ Benchmark script could divide by zero (`ELAPSED_S` rounds to 0.0 on small restores — verified `bc` aborts). Throughput now computed from milliseconds with a zero guard.
- ✅ Smoke test hard-failed on a non-standard `admin` account — now marked "replace with an account in YOUR environment".
- ✅ Bareos client name aligned to `backup-client-1` (module 07's resource name) in both places it differed.

### Module 11 — security & compliance
- ✅ `semanage fcontext -a -t backup_t …` — no `backup_t` type exists in the RHEL targeted policy; the command errors. Fixed to `var_t` with a note about custom types.
- ✅ `chattr +i /backup/` (the live backup root) would break the module's own backup jobs; `chattr +a` on a restic `data/` dir breaks `restic prune` (which the module also recommends). Examples retargeted to a sealed archive copy / log dir, with an explicit warning and pointer to rest-server `--append-only` / object locking for restic.
- ✅ Object-lock example used GOVERNANCE mode as ransomware protection — bypassable via `s3:BypassGovernanceRetention`. Switched to COMPLIANCE with explanation.
- ✅ `grep -rn "password\s*=…"` used `\s` without `-P` (matches literally in basic grep) and a useless `grep -v "^Binary"`. Fixed to `grep -rnPI`.
- ✅ Lab GPG key used `ECDSA/nistp256` while §2.1 recommends Curve 25519 — lab now generates `ed25519`/`cv25519`.
- ✅ `ssh-keygen -lf` sample output showed a username (host-key fingerprints show the host); restic scrypt wording tightened (master key is random, wrapped by the scrypt-derived key) — matching the same fix in module 06.

### Module 12 — capstone
- ✅ Inventory heredoc used a quoted delimiter, writing literal `$(date …)` placeholder lines that then got duplicated by the echo block. Fixed (header-only + echos).
- ✅ Disaster Method B ran `dd if=/dev/zero of=/dev/sda` with no check that `/dev/sda` is the OS disk — on VirtIO VMs root is often `/dev/vda` and `/dev/sda` may be the **backup** disk. Added a mandatory pre-flight (`lsblk`, `findmnt -no SOURCE /`, compare with inventory).
- ✅ `chmod 700 /backup` (root-owned) made every subsequent `scp`/sftp/rsync as `backupuser` fail. Fixed (backupuser ownership, 750).
- ✅ xfsdump step guarded on `/dev/backupvg/data` + `/mnt/data` — the shared env creates `datalv` mounted at `/data`, so the step always skipped. Fixed.
- ✅ Rescue-shell restic download: dead `.bz2`-then-overwrite-with-404 sequence. Fixed (pinned version, single download, `bunzip2 -c`).
- ✅ Rescue network setup assumed `eth0` + `dhclient` (neither exists on RHEL 10 media). Fixed (`ip link` discovery, static `ip addr`).
- ✅ Chroot step: fstab fix was a passive comment — restored fstab references old UUIDs and the not-yet-recreated `backupvg`, which hangs first boot. Now an explicit numbered step. UEFI `grub2-mkconfig` target corrected to `/boot/efi/EFI/redhat/grub.cfg`.
- ✅ Extension D runbook rebuilt the bootloader BIOS-style (`grub2-install $TARGET_DISK`) although the whole scenario is UEFI. Made UEFI-consistent (EFI mount + package reinstall), BIOS noted as the alternative.
- ✅ Canary verification diffed mismatched file sets (and caught the section header via `grep CANARY`), guaranteeing spurious failures on a perfect restore. Both sides now enumerate the identical set.
- ✅ Phase 5 timing: `RESTIC_SYSTEM_UP` was echoed locally but never written to the server's timing file, and `TAR_SYSTEM_UP`/`RSYNC_SYSTEM_UP` were read but never recorded. Fixed (ssh-append; B5/C3 record their own keys; analysis script accepts per-scenario start keys).
- ✅ Pre-lab checklist now names the exact SSH key path (`/root/.ssh/backup_ed25519`) used by later commands (earlier modules created differently-named keys).

### Modules 00, 01, README
- ✅ `01` Answer 9: `/var/spool/cron/crontabs/` is the **Debian** layout; RHEL is `/var/spool/cron/<user>` (the answer even contradicted its own next sentence). Fixed.
- ✅ `01`: dropped non-existent `/var/lib/mariadb/` path (RHEL uses `/var/lib/mysql/`); EFI note now says VFAT and points at the real UEFI grub.cfg location; Lab 01-4's `file`/`df -T` commands (which don't demonstrate "virtual filesystem") replaced with `findmnt`/`mount | grep`.
- ✅ `01` §"Database warning" pointed to "Module 09 for details" — module 09 never covers DB-native dumps. Re-pointed to the actual tools + Module 05 quiescing.
- ✅ `README`: documented Module 12 as the intentional exception to the "every module ends with Review Questions + Answers" claim; softened "`/dev/sdb` used throughout".
- 📋 `00-introduction.md`: no factual errors found. Minor wording-only items (crash-consistency "Method" column over-narrowed to LVM snapshots; answers 4/10 slightly broader than the body) left as-is.

---

## Currency updates applied

- ✅ restic pinned version `0.17.3` → `0.18.1` (current; matches EPEL 10) in modules 06 and 12.
- ✅ restic `self-update` now warns it's only for the manual `/usr/local/bin` install, never the RPM.
- ✅ Bareos EL_10, Amanda EPEL-10 absence, `dump` absence, `s-nail` — see tables above.

## Pedagogy improvements applied

- ✅ M02: note that the `.snar` file is required state for continuing an incremental chain (protect/back it up); "fresh `.snar` by copying or removing" corrected (copying does not reset it).
- ✅ M05: added the crash-consistent vs application-consistent distinction (the module's premise oversold raw snapshots as solving DB consistency) and the standard `snapshot_autoextend_threshold`/`dmeventd` auto-extend safety net; `lvconvert --merge` "requires reboot" qualified (only while the origin is in use).
- ✅ M06: added the **"lost password = permanently unrecoverable data"** warning and the scrypt key-wrapping detail.
- ✅ M04: warning before restore-over-live-root; cross-reference to freeze/snapshot at the first live-dump example.

---

## Recommendations — resolution status (second pass, same day)

All items below were resolved in a follow-up pass; decisions marked *(author)* were confirmed with the course author.

1. ✅ **Module 05 rewritten to the shared lab environment** *(author)* — examples and labs now snapshot `backupvg/datalv` (`/data`, 1G snapshots fitting the ~2G free VG space); thin-pool section resized (1G pool, 5G over-provisioned thin LV — used as a teaching point with warning); one clearly-marked `/dev/rhel/root` system-VG example retained; §9 DB examples annotated as production examples; labs, Q10/A10, and the attr-decode gloss all updated.
2. ✅ **Retention standardised on 12 monthly / 7 yearly** *(author)* — module 06's five `--keep-monthly 6 --keep-yearly 1` occurrences updated to match module 09.
3. ✅ **Prompt-convention split documented in README** *(author)* — conventions section now explains both styles: `sudo` blocks in tool modules 00–08, prompt markers in scenario modules 09–12.
4. ✅ Stray hostnames `web01`/`db01` replaced with `backup-client`/`backup-server` in modules 02 and 06.
5. ✅ Module 12 restic repo converted to the per-host layout `/backup/restic/backup-client` (literal hostname rather than `$(hostname)` so the path also works from a rescue shell).
6. ✅ Module 07: Director Storage `Address` changed from `localhost` to `backup-server` with a warning (clients connect to the SD at this address); `File = /data` added to `DefaultFileSet`; `FileStorageIncremental` device annotated (needs a matching Director Storage resource; given its own `Media Type = FileIncr`); §17.9 empty code block converted to prose; `update bvfs` → `.bvfs_update`; `list jobs last=7` comment clarified (job count, not days). *(Still open: verify the 13-item restore-menu listing and `make_catalog_backup.pl` output filename against a live Bareos install — both need a running system to confirm.)*
7. ✅ Module 03: exclude list trimmed to patterns that can actually match the chosen sources (with an explanatory comment); push example switched from `root@` to `backupuser@`.
8. ✅ Module 09: PID-file lock replaced with atomic `flock` on an FD (kernel-released, no stale locks). *(The tee/journal double-logging note was deemed acceptable redundancy and left.)*
9. ✅ Module 11: auditd now watches both `/usr/bin/restic` and `/usr/local/bin/restic` in both rule sets.
10. ✅ Module 02: `-d`/`--compare` added to the flags reference table; useless-use-of-cat in the snar inspection fixed.
11. ✅ Module 00: Answer 4 aligned with the body's "unless versioned" nuance; crash-consistent "Method" column generalised to "snapshot without quiescing".

---

## Verification performed

- Link/anchor checker over all 15 files: **0 broken** (was 17 + 1 introduced-and-fixed during editing).
- Suspected shell bugs reproduced before fixing and re-tested after (`echo \n`, IFS/exclude collapse, comment-after-backslash, `bc` divide-by-zero).
- Package availability confirmed against this RHEL 10 host's repos (`dnf info/repoquery`): `dump` ✗, `amanda*` ✗, `mailx` ✗, `s-nail` ✔, `restic 0.18.1` ✔, `xfsdump` ✔, `fuse` 2.9.9 ✔, `fuse3` ✔.
- Bareos EL_10 repo existence confirmed at download.bareos.org; restic/amanda EPEL status confirmed via packages.fedoraproject.org.
- `git diff` reviewed hunk-by-hunk against this report.

*This file is a review artifact — exclude it from the published course or add it to `.gitignore` if it should not ship.*
