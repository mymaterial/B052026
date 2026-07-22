# ACID Properties & VM Lab Setup — Session Notes

This session covers how PostgreSQL actually implements each ACID property — atomicity via `BEGIN`/`COMMIT`, isolation via MVCC (with a live row-versioning demo), durability via WAL — including a detailed discussion of why WAL has no Oracle-style redo-group redundancy and what that means in practice. The second half walks through setting up and cloning the VM lab environment for a new batch.

---
## Table of Contents

- [1. Atomicity — BEGIN and COMMIT](#1-atomicity--begin-and-commit)
- [2. Consistency — No Fixed Definition](#2-consistency--no-fixed-definition)
- [3. Isolation — MVCC](#3-isolation--mvcc)
- [4. MVCC vs. Oracle's Undo Tablespace](#4-mvcc-vs-oracles-undo-tablespace)
- [5. When MVCC Row Bloat Becomes a Real Problem](#5-when-mvcc-row-bloat-becomes-a-real-problem)
- [6. Durability — WAL](#6-durability--wal)
- [7. No Redo Multiplexing — Only Archive Multiplexing](#7-no-redo-multiplexing--only-archive-multiplexing)
- [8. The One Genuinely At-Risk File](#8-the-one-genuinely-at-risk-file)
- [9. VM Lab Setup and Cloning](#9-vm-lab-setup-and-cloning)
- [10. What's Next](#10-whats-next)
- [11. Questions Discussed in This Session](#11-questions-discussed-in-this-session)

---

## 1. Atomicity — BEGIN and COMMIT

**Definition given, with the classic money-transfer example:** a transaction may consist of multiple statements (debit account A, credit account B) — atomicity means treating them as one indivisible unit: either **all** statements in the transaction apply, or **none** do. It should never be possible for the debit to succeed and the credit to fail, leaving the system in an inconsistent state.

```sql
BEGIN;
UPDATE employee SET salary = 600 WHERE id = 1;   -- succeeds
UPDATE employee SET syntax_error = 700;           -- fails
-- PostgreSQL response: "current transaction is aborted, commands ignored until end of transaction block"
COMMIT;   -- has nothing to commit; the whole block rolls back
```

**Demonstrated live:** once any statement inside a `BEGIN` block fails, PostgreSQL aborts the **entire transaction** — even statements that already ran successfully within that same block get rolled back once `COMMIT` is issued (or the session ends) after a failure. `BEGIN`/`END` (or `BEGIN`/`COMMIT` — the two are functionally equivalent) is how PostgreSQL achieves atomicity.

**Practical consequence, repeated from earlier sessions:** PostgreSQL auto-commits by default. Without wrapping a real production change in `BEGIN`, there is no rollback available if something goes wrong — recovery at that point means point-in-time recovery, which is a much heavier operation on a large database.

---

## 2. Consistency — No Fixed Definition

**Stated directly:** unlike the other three ACID letters, PostgreSQL (and the ACID literature generally) doesn't have one crisp technical definition for "consistency." The practical framing given: as long as atomicity holds, the database's before-and-after states for any transaction should themselves be internally consistent — e.g. in the money-transfer example, the sum of both balances should be identical before and after a successful transfer; a failed transaction should leave the pre-transaction state completely undisturbed.

---

## 3. Isolation — MVCC

**Definition given:** reads and writes should not interfere with each other — a reader should always see a *consistent past image* of the data, never a partial or in-progress write from another session.

**Live demo:**
```sql
-- Session 1
SELECT * FROM employee;   -- id=1, salary=1000
BEGIN;
UPDATE employee SET salary = 2000 WHERE id = 1;   -- not yet committed
```
```sql
-- Session 2, concurrently
SELECT * FROM employee;   -- still shows salary=1000
```
The second session sees the old, committed value — the in-progress change in Session 1 is invisible until it's committed.

**What's happening internally, walked through step by step:**
1. Before any modification, the existing row (e.g. `salary=1100`) is left untouched — reads continue to see this version.
2. On `UPDATE`, PostgreSQL creates a **new copy of the row** with the change applied, marked as an in-progress ("stormed"/uncommitted) version, invisible to any other session.
3. While the transaction is open, **two versions of the same logical row exist simultaneously**: the old, visible-to-everyone-else version, and the new, visible-only-to-this-session version.
4. On `COMMIT`, the new version becomes the current, generally visible version; the old version is marked **dead** (a dead tuple) — cleaned up later by autovacuum/vacuum (covered in depth in earlier sessions).

**This is the definition of MVCC given directly:** *PostgreSQL maintains multiple versions of the same record while a transaction is in progress, in order to satisfy the isolation property.*

**Confirmed live — reads never block writes, and vice versa, but write-write conflicts do block:** a second session's `SELECT` succeeded without waiting while a `BEGIN`-wrapped `UPDATE` was still open elsewhere. But attempting to `UPDATE` **the same row** from a second session while the first session's transaction was still open on it **blocked** — confirmed as PostgreSQL's row-level exclusive lock, which prevents two concurrent writers from modifying the exact same row simultaneously (this is expected, correct locking behavior, not an isolation failure — isolation concerns reads-vs-writes, not concurrent writers on the same row).

---

## 4. MVCC vs. Oracle's Undo Tablespace

**Direct comparison drawn for anyone with an Oracle background:** Oracle achieves isolation via undo tablespace — the *old* version of a changed row is copied out to undo, and readers needing the pre-change image read it from there while the transaction is in flight. Once committed, the undo entry becomes obsolete and is eventually overwritten.

**A specific Oracle failure mode does not exist in PostgreSQL:** Oracle's shared undo tablespace can suffer "snapshot too old" errors when a large, long-running transaction's undo entries get overwritten by other concurrent transactions needing undo space. **PostgreSQL doesn't have a shared undo pool at all** — each table effectively manages its own old-row versions in place (via MVCC's copy-on-write row versioning), so there's no shared, finite undo resource that a different transaction can exhaust or evict from underneath a long-running one.

---

## 5. When MVCC Row Bloat Becomes a Real Problem

**A worked hypothetical, explicitly labeled as rare:** repeatedly updating the same large table (e.g. 128MB) without autovacuum ever running to reclaim dead tuples means each `UPDATE` leaves the old version behind as dead weight — four successive updates on a never-vacuumed 128MB table could theoretically balloon it toward 512MB+ of mostly-dead row versions.

**Explicitly qualified:** this scenario is described as something that "rarely" actually happens in practice, on properly configured production systems where autovacuum is running as expected — it's presented here purely to make the underlying MVCC mechanism visible and concrete, not as a realistic day-to-day production risk.

---

## 6. Durability — WAL

**Definition given:** at any cost, the database (or at minimum, the *data*) must be safe/recoverable.

```bash
ls /u01/pgsql18/pg_wal/
```
Every change made to the database is first recorded here (the transaction/WAL log). As long as these files are available, the underlying data can theoretically be reconstructed even if the actual data files are lost.

**Explicitly qualified — recoverable is not the same as easy:** recovering data purely from WAL files is described as genuinely difficult in practice — cited as one of the most common answers when DBAs are asked "what's the hardest problem you've solved" (point-in-time recovery, ~90% of the time per the instructor's informal poll of past experience). PostgreSQL achieves durability via **WAL (Write-Ahead Logging)**.

---

## 7. No Redo Multiplexing — Only Archive Multiplexing

**A direct architectural contrast with Oracle, worked through carefully:**

- **Oracle has redo log groups** — the same redo information is written to multiple mirrored members simultaneously, so a single corrupted redo file doesn't lose that information.
- **PostgreSQL has no equivalent "WAL group" concept.** Each WAL segment is a single file (default 16MB); there is no built-in mechanism to write the *same* active WAL segment to multiple redundant locations simultaneously.

**What you *can* do instead — multiplex the archive, not the active WAL file:**
```ini
archive_command = 'cp %p /archive_location_1/%f && cp %p /archive_location_2/%f'
```
Once a WAL segment fills and rolls over, it gets archived — and the archive step can copy that segment to **multiple** archive destinations, giving you redundant copies of every *completed* WAL segment. This is functionally similar to Oracle's redundancy, just applied after the fact rather than during active writing.

**Practical guidance:** archive to more than one location (different mount point, and ideally a different physical server) as standard practice — not because it's mandatory, but because it's the only real redundancy mechanism available for WAL data in PostgreSQL.

---

## 8. The One Genuinely At-Risk File

**The specific gap called out directly:** the **currently active** WAL segment — the one being written to right now, before it fills and rolls over to archiving — has **no redundant copy anywhere**. If that specific file is lost or corrupted before it completes and archives, that window of transactions has no backup copy. This is presented as an inherent, unavoidable architectural characteristic of PostgreSQL's WAL design, not a misconfiguration to fix — mitigated by keeping the active WAL segment size reasonable (16MB default) and by ensuring good general reliability of the storage the active WAL directory sits on, since it can't be multiplexed.

**Also clarified:** backing up WAL files via a dedicated backup *tool* the way Oracle has RMAN for archived redo logs doesn't have a direct PostgreSQL equivalent — the practical approach is OS-level copy/`rsync` of archived WAL files to a backup location, or (the cleaner mental model recommended) simply treating a second archive destination as "the backup copy of the first" rather than thinking of it as a separate backup step.

---

## 9. VM Lab Setup and Cloning

**Base VM setup process walked through live:**
1. Download the pre-built base VM image (`lab01.zip`) from a shared location.
2. Extract it, right-click the `.vmx` file, open with VMware Workstation (VirtualBox users can continue using VirtualBox — the tool choice doesn't matter).
3. On first boot, choose **"I copied it"** (not "I moved it") when prompted — this regenerates the VM's unique network identity rather than treating it as the original.
4. Log in as `root` / the shared lab password; confirm networking with `ip addr` / `ifconfig` and `ping google.com`.

**Cloning an existing VM (e.g. to get a second lab node):**
1. **Stop the source VM first** — this is a mandatory step; cloning a running VM's folder isn't supported.
2. Copy the entire VM folder (e.g. `lab01` → `lab02`).
3. **Do not** open the copied `.vmx` file directly with VMware Workstation the first time — open it with a plain text editor first, find the `displayName` field, and change it to the new machine's intended name, to avoid confusing it with the source VM in the VMware UI.
4. *Then* open it with VMware Workstation normally, confirm **"I copied it,"** and power on.
5. **Change the hostname** — since the clone still reports the original machine's hostname:
```bash
hostnamectl set-hostname lab02
reboot
```

**Networking troubleshooting — VMware's Virtual Network Editor:**
- Path: **Edit → Virtual Network Editor → Change Settings → Restore Defaults.**
- **Only needed in two specific situations:** the VM isn't getting an IP address at all, or it has an IP but no internet access. It is explicitly **not** something to reach for just because you'd prefer a different subnet range — the default `192.168.x.128/24`-style range (exact third/fourth octet varies by machine) is fine to just use as assigned in the vast majority of cases.

**A separate, lighter-weight use of "cloning": snapshotting your own progress.** Rather than re-doing a long or fiddly setup (the example given: installing `ora2pg`, which can take 2–3 hours), copy the whole VM folder to a new location once that setup is complete, and rename the folder for what it represents (e.g. a `patroni` parent folder containing `test01`/`test02`/`test03`) — a cheap way to preserve a known-good state without repeating time-consuming setup work.

---

## 10. What's Next

- An optional Linux-basics session offered the same evening for anyone from a non-Linux (e.g. SQL Server) background, covering `cp`, `tar`, `dnf`/`yum`, `wget`, `cat`, `tail`, `find`, `ps`, `free`, `df`, `iostat`/`sar` — attendees already comfortable with Linux can skip it.
- Actual PostgreSQL installation begins the following session (~3 days planned), building the OS from an ISO image with custom partitioning rather than relying on a pre-built snapshot going forward — the pre-cloned VM approach shown today was specifically to get people set up quickly for the interim.

---

## 11. Questions Discussed in This Session

**Q1. If a WAL file is lost or corrupted before archiving, does the database stop operating?**

No — the running instance continues to operate normally. What's lost is the *ability to use that specific segment's transactions for point-in-time recovery* later, not the live database's current operation. The practical mitigation is redundancy at the archive level (Section 7), since the currently-active WAL segment itself cannot be mirrored the way Oracle's redo groups are.

---

**Q2. During point-in-time recovery, does PostgreSQL read from the live WAL directory first, then fall back to the archive location — or only from the archive?**

Recovery works from the **archive location**, not the live WAL directory — WAL segments are archived as they roll over, and it's from that archived sequence of segments that point-in-time recovery replays transactions forward (or to a specific target point). There's no "check the live directory first" step in that process.

---

**Q3. Can PostgreSQL's WAL files be backed up the way Oracle's archived redo logs are backed up with RMAN?**

Not with a dedicated backup tool built for that specific purpose the way RMAN handles archived redo — the practical equivalent is an OS-level copy or `rsync` of archived WAL segments to a secondary location, or (the cleaner approach recommended) configuring `archive_command` to write to multiple archive destinations directly, treating the second location as the de facto backup of the first rather than a separate backup process.

---

**Q4. Does concurrent write access to the same row genuinely block, or does MVCC allow simultaneous writes too?**

MVCC's multi-versioning specifically protects **reads** from being blocked by an in-progress write — it does not allow two concurrent writers to modify the exact same row simultaneously. Confirmed live: a second session attempting to `UPDATE` a row already open for update (uncommitted) in another session blocked on PostgreSQL's row-level exclusive lock, exactly as expected — this is standard, correct write-write serialization, separate from (and not a violation of) the isolation property, which concerns read-vs-write interference specifically.
