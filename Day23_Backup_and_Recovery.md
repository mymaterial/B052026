# Physical Backup Internals, Point-in-Time Recovery & Streaming Replication — Session Notes

This is a long, hands-on session (~3 hours) building up from first principles: what `pg_basebackup` actually does internally, how to restore a plain backup, how to replay archived WAL after restoring (recovery), how to stop recovery at a specific point (PITR), how to keep a standby continuously catching up (standby mode), and finally how `pg_basebackup -R` turns all of that into a one-command streaming replication setup — ending with converting asynchronous ("maximum performance") replication into synchronous ("maximum protection") replication.

---
## Table of Contents

- [1. What Happens Internally During pg_basebackup](#1-what-happens-internally-during-pg_basebackup)
- [2. The backup_label File](#2-the-backup_label-file)
- [3. Lab Setup — Three Machines](#3-lab-setup--three-machines)
- [4. Passwordless Connectivity and Remote Backups](#4-passwordless-connectivity-and-remote-backups)
- [5. Task 1 — Plain Backup and Restore](#5-task-1--plain-backup-and-restore)
- [6. Task 2 — Recovering Archived WAL (recovery.signal)](#6-task-2--recovering-archived-wal-recoverysignal)
- [7. Task 3 — Point-in-Time Recovery](#7-task-3--point-in-time-recovery)
- [8. Task 4 — Standby Mode (standby.signal)](#8-task-4--standby-mode-standbysignal)
- [9. How Far Behind Can a Standby Be?](#9-how-far-behind-can-a-standby-be)
- [10. Task 5 — Streaming Replication with pg_basebackup -R](#10-task-5--streaming-replication-with-pg_basebackup--r)
- [11. The Data-Loss Problem With Async Replication](#11-the-data-loss-problem-with-async-replication)
- [12. Task 6 — Synchronous Replication](#12-task-6--synchronous-replication)
- [13. What's Next](#13-whats-next)
- [14. Questions Discussed in This Session](#14-questions-discussed-in-this-session)

---

## 1. What Happens Internally During pg_basebackup

Before running any commands, the internal sequence was worked through conceptually:

```
1. Issue a CHECKPOINT (or wait for the next one) — records "backup starts here"
2. Log switch — start of backup, marked in a new WAL file
3. Copy every database's data files to the backup location, one by one
   (the source database keeps running and generating new transactions
    the entire time this copy is happening)
4. Log switch again — end of backup
5. Copy all WAL files generated *during* the backup into the backup location too
```

**Why WAL files generated during the backup have to be included:** while data files are being copied (which can take a long time on a large database), the live system keeps generating new transactions. Those changes exist only in WAL until a later checkpoint flushes them to disk — so a backup consisting of data files alone, without also capturing the WAL generated during the copy, would be internally inconsistent. Restoring later replays those WAL files to bring the copied data files up to a consistent state.

**Practical verification:** after taking a backup, the WAL files present in the backup's own `pg_wal` directory correspond exactly to the range generated between the backup's start and end — confirmed live by checking WAL filenames against the archive location's timestamps during the demo.

---

## 2. The backup_label File

When a backup starts, PostgreSQL writes a `backup_label` file into the backup's data directory. This file records which WAL file the backup started at.

**On startup, if PostgreSQL finds a `backup_label` file**, it knows this is a freshly restored backup (not a normal restart) and cross-checks WAL starting from the recorded position — rather than trusting whatever the latest checkpoint claims — because the data files may be from an earlier, inconsistent point relative to the last checkpoint recorded in the control file. Once that startup is complete, PostgreSQL renames the file to mark this cross-check as done, so subsequent normal restarts don't repeat it.

**If you manually delete `backup_label` before starting a restored backup**, PostgreSQL has no way to know it's looking at a restored backup — it treats the data directory as an ordinary shutdown/restart and only cross-checks from the latest checkpoint in the control file, potentially skipping WAL it actually needs to apply. **This is a real, if less common, misconfiguration** worth watching for — deleting "extra-looking" files from a restored backup directory without understanding what they're for can silently corrupt the restore.

A brief demo also showed the **manual/low-level backup mode** (calling the start/stop backup functions directly, in one session, while copying files in a second session) purely to illustrate the mechanism — explicitly flagged as **never used in real production systems**; `pg_basebackup` (or a backup tool like PGBackRest) is the actual production method.

---

## 3. Lab Setup — Three Machines

Three fresh VMs (`Lab01`, `Lab02`, `Lab03`) were provisioned for this and upcoming replication sessions:

```bash
# on each machine
useradd postgres
passwd postgres
# enable EPEL / PGDG repos, install PostgreSQL as covered in earlier installation sessions
```

```bash
hostnamectl set-hostname Lab02   # run on the 2nd machine
hostnamectl set-hostname Lab03   # run on the 3rd machine
reboot
```

- `Lab01`: `192.168.108.131` — first instance initialized here, archive logging enabled (`archive_mode = on`, `archive_command` copying WAL to `/u01/archive_logs`).
- `Lab02`: `192.168.108.132`
- `Lab03`: `192.168.108.133`

PostgreSQL was installed on all three machines, but the cluster was only initialized on `Lab01` — `Lab02` and `Lab03` start empty, to be populated via backup/restore and replication in the sections that follow.

---

## 4. Passwordless Connectivity and Remote Backups

```bash
# on Lab01, as the postgres OS user
ssh-keygen
ssh-copy-id postgres@192.168.108.132   # Lab01 -> Lab02
ssh-copy-id postgres@192.168.108.131   # (reverse direction set up too, from Lab02)
```

**Why:** WAL files need to be shipped between machines without a human typing a password each time (via `archive_command`/`restore_command` using `scp`), and later, replication connections need to be established automatically — both require passwordless SSH.

**Taking a remote backup — pg_hba.conf needs a `replication` entry:**

```
# on Lab01's pg_hba.conf
host   replication   all   192.168.108.132/32   md5
```

```bash
pg_ctl reload -D /u01/pgsql18
```

```bash
# run from Lab02, pulling a backup of Lab01
/usr/pgsql-18/bin/pg_basebackup -D /u01/pgsql18 --checkpoint=fast -P -h 192.168.108.131
```

**Key distinction clarified:** a `replication`-type `pg_hba.conf` entry is only needed when taking a backup **from a remote machine**. Backing up a local instance (`pg_basebackup` run on the same machine the database is running on) never needs a replication connection — it uses a normal local connection.

---

## 5. Task 1 — Plain Backup and Restore

The simplest case: take a backup, then just start it as-is.

```bash
pg_basebackup -D /u01/pgsql18 --checkpoint=fast -P -h 192.168.108.131
pg_ctl start -D /u01/pgsql18
```

**There is no separate "restore" command in PostgreSQL** — restoring is simply: stop the instance (if running), replace/point the data directory at the backed-up files, and start. PostgreSQL figures out what state it's in from the data directory's contents (control file, `backup_label` if present, WAL files) rather than being told explicitly what to do.

**Limitation of Task 1:** this restores the database exactly to the moment the backup finished — any transactions committed *after* the backup completed on the source are simply not present. This is the baseline every other task in this session improves on.

---

## 6. Task 2 — Recovering Archived WAL (recovery.signal)

**Goal:** after restoring, also replay every archived WAL file generated *after* the backup finished, to bring the restored copy fully up to date with everything archived so far.

```bash
touch /u01/pgsql18/recovery.signal
```

```ini
# postgresql.conf
restore_command = 'scp postgres@192.168.108.131:/u01/archive_logs/%f %p'
```

`restore_command` is the mirror image of `archive_command` — instead of copying a WAL file *out* to the archive location, it copies a requested WAL file *in* from the archive location, using `%f` (the requested filename) and `%p` (the destination path PostgreSQL wants it placed at).

**On startup, PostgreSQL sees `recovery.signal` and knows to actively pull and replay archived WAL** (rather than just relying on whatever's already in its own `pg_wal` directory). It applies every WAL file it can find in the archive, in order, until it hits one that isn't there yet — at which point (with only `recovery.signal`, no target set) it stops replaying, assumes recovery is complete, and opens the database on a **new timeline ID**.

**Confirmed live:** after taking a backup, generating additional data + archived WAL on the source, then restoring with `recovery.signal` + `restore_command`, all the post-backup WAL was replayed and the additional data was present in the restored copy.

---

## 7. Task 3 — Point-in-Time Recovery

Same setup as Task 2, but stopping recovery deliberately at a specific point rather than replaying everything available — for the classic "someone dropped a table around 10:15 AM, restore to just before that" scenario (formally introduced toward the end of the session as the motivating use case for this whole topic).

```ini
restore_command = 'scp postgres@192.168.108.131:/u01/archive_logs/%f %p'
recovery_target_lsn = '<specific LSN>'
# (recovery_target_time and recovery_target_name are the other common alternatives
#  to recovery_target_lsn, for stopping at a timestamp or a named restore point instead)
```

With a `recovery_target_*` parameter set, PostgreSQL replays WAL only up to that point, then stops and opens the database — deliberately leaving out anything that happened after the target, exactly like Task 2 but bounded instead of "replay everything available."

---

## 8. Task 4 — Standby Mode (standby.signal)

**Difference from Task 2:** instead of stopping once it runs out of available archived WAL, keep checking for new WAL and keep applying it as it arrives — i.e., behave as an ongoing standby rather than a one-time recovery.

```bash
touch /u01/pgsql18/standby.signal
```
(same `restore_command` as before)

**Observed behavior:** recovery applies every archived WAL file it can find, and when it hits one that isn't there yet, it does **not** give up and open the database — it waits, and retries every **5 seconds**, until that WAL file eventually shows up in the archive location.

**A subtlety demonstrated live:** a `CHECKPOINT` on the primary does **not** by itself force a WAL file switch — a checkpoint just records its position within whatever WAL file is currently being written. A new transaction alone also doesn't force a switch if it fits in the current (still-open) WAL file. The standby only picks up new data once the primary's WAL segment actually rolls over — either because it filled up, or because something explicitly forced a switch (`SELECT pg_switch_wal();`).

---

## 9. How Far Behind Can a Standby Be?

With the Task 4 setup (archive-based standby, default settings, no `archive_timeout` set):

- The default WAL segment size is **16MB**. A WAL file isn't archived (and therefore isn't available for the standby to pick up) until it's full — or until something forces an early switch.
- **If the primary system is idle**, WAL simply doesn't fill up, and no switch happens — there's no PostgreSQL equivalent of "switch logs automatically every N minutes" by default (unlike some other databases' log-switch-interval settings).
- **Practical consequence:** in this setup, a standby can lag the primary by **up to 16MB of unshipped WAL**, indefinitely, if the workload is light enough that a WAL file takes a long time to fill.

**`archive_timeout` closes this gap on a time basis instead:**

```ini
archive_timeout = 60   # force a WAL switch at least every 60 seconds
```

This bounds the maximum data-loss/lag window to roughly 1 minute of activity instead of "however long it takes to fill 16MB" — at the cost of generating (and archiving) a WAL file every 60 seconds even during genuinely idle periods, which is wasted archive storage on a quiet system. This is a real trade-off to make deliberately, not a "just turn it on" default.

---

## 10. Task 5 — Streaming Replication with pg_basebackup -R

Everything from Tasks 1–4 was built manually to understand the mechanism. `pg_basebackup -R` automates the standby setup in one step:

```bash
pg_basebackup -D /u01/pgsql18 --checkpoint=fast -P -h 192.168.108.131 -R
pg_ctl start -D /u01/pgsql18
```

**What `-R` actually does, confirmed by inspecting the resulting files:**
1. Creates `standby.signal` automatically.
2. Writes `primary_conninfo` (host, username, password) into `postgresql.auto.conf` automatically, pointing back at the machine the backup was taken from.

This is genuinely **streaming** replication — the standby connects directly to the primary and receives changes as they happen, rather than only picking up whatever's been archived to shared storage. (`restore_command` can still be layered on top as a fallback for WAL the standby missed while briefly disconnected, but the live streaming connection is the primary path.)

**Verified live:**
- Writes on the primary (`INSERT`) appeared on the standby almost immediately.
- Write attempts directly on the standby (`DELETE FROM employee;`) were rejected — **a standby is read-only**, by design, as long as `standby.signal` is present.
- Stopping the standby, making changes on the primary, then restarting the standby: the standby correctly caught back up once restarted, replaying what it missed.
- **If the primary machine is destroyed entirely** while the standby was stopped (not just paused), any changes made on the primary *after* the standby last synced are permanently lost — the standby only ever has what it received before it fell behind; there's no way to recover data that only ever existed on the now-gone primary. This is exactly the scenario that motivates synchronous replication (Section 12).

---

## 11. The Data-Loss Problem With Async Replication

The setups in Tasks 4–5 are all **asynchronous** by default — the primary commits a transaction and reports success to the client *without waiting* for the standby to confirm it received that change. This is good for performance (client latency isn't tied to network round-trips to a standby) but means a standby is always potentially some amount behind, and if the primary is lost at exactly the wrong moment, that gap of unreplicated data is gone permanently.

This maps to the same tradeoff Oracle DBAs will recognize from Data Guard:

| Oracle Data Guard term | Equivalent here |
|---|---|
| Maximum Performance mode | Default asynchronous streaming replication (Tasks 4–5) |
| Maximum Protection mode | Synchronous replication (Task 6) |
| Maximum Availability mode | PostgreSQL's cascading replication setup (mentioned as a preview only — not covered in depth this session) |

---

## 12. Task 6 — Synchronous Replication

Converts the setup from "maximum performance" to "maximum protection": the primary will not report a transaction as committed until at least one synchronous standby has confirmed it received the change.

**On the standby**, add an application name to its connection info so the primary can identify it specifically:

```ini
# postgresql.auto.conf (or manually added to primary_conninfo) on the standby
# primary_conninfo already has host/user/password from -R; add:
application_name = standby1
```

**On the primary**, declare which standby (by that application name) must confirm before a commit is considered durable:

```ini
# postgresql.conf on the primary
synchronous_standby_names = 'standby1'
```

```bash
pg_ctl reload -D /u01/pgsql18   # on the primary
# restart required on the standby side for primary_conninfo/application_name changes
```

**Note on a loose end flagged at the very end of the session:** `max_wal_senders` (the parameter controlling how many concurrent replication connections a primary will accept) was noticed as never having been explicitly set during this walkthrough — left at its default. This was flagged as something to come back to and verify is adequate, not something demonstrated as broken.

---

## 13. What's Next

Explicitly scheduled as continuations rather than covered in this session:

- Point-in-time recovery walked through as a realistic incident-response scenario (a table dropped sometime in a known window, without an exact timestamp) — ~10–15 minutes of dedicated time planned.
- **Hot standby conflicts** — situations where read queries on a standby conflict with WAL being applied — ~15–20 minutes planned.
- Streaming replication corner cases, and sync-check commands for verifying replication health.
- **Cascading replication** — `Lab01` replicating to `Lab02`, `Lab02` in turn replicating to `Lab03`.
- After on-prem topics are finished: moving to AWS, then covering manual switchover/failover, followed by pooling/HA tooling (PgBouncer, PgPool, Patroni) as tools built on top of these same underlying concepts.

**Teaching note given directly:** the intent of these sessions is to cover the underlying concepts thoroughly (~90% of what's needed), not to walk through every minor SOP-level command variation. Students were encouraged to search for supplementary how-to guides on topics already covered (e.g. "how to set up streaming replication in PostgreSQL") specifically to cross-check for any small step the class might have glossed over — and to flag anything that appears to genuinely *contradict* what was taught in class, as opposed to simply covering additional minor detail.

---

## 14. Questions Discussed in This Session

**Q1. If I'm taking a backup from within the same machine the database is running on, do I still need a `replication`-type entry in `pg_hba.conf`?**

No — a replication connection is only required when `pg_basebackup` is connecting to a **remote** machine (`-h` pointing at a different host). A local backup uses an ordinary local connection and works with the standard `local`/`host` rules already in `pg_hba.conf`.

---

**Q2. Is a streaming-replication standby (Task 5) ever guaranteed to be fully in sync with the primary?**

No, not by default — the setup in Tasks 4–5 is asynchronous. The primary doesn't wait for standby confirmation before reporting a commit as successful, so the standby can always be some amount behind (bounded by WAL segment size and `archive_timeout`, if set — see Section 9). Only synchronous replication (Task 6) gives a guarantee that a committed transaction is also present on at least one standby before the client sees "commit successful."

---

**Q3. Is there a PostgreSQL equivalent to Oracle's log-switch-interval / lag-target parameter, forcing periodic WAL switches even on an idle system?**

`archive_timeout` — set it to force a WAL switch at least every N seconds, bounding the maximum replication lag/data-loss window to roughly that interval instead of "until 16MB of WAL accumulates." It's not on by default, and turning it on means generating (and archiving) WAL files even during genuinely idle periods, which is a real storage/activity tradeoff to weigh, not a free improvement.

---

**Q4. In `pg_basebackup -R`, how does the tool know which specific primary instance to connect to for streaming replication, since only `-h` was specified?**

`-R` writes `primary_conninfo` into `postgresql.auto.conf`, and the connection details it records (host, port, username, password) come directly from the `-h` (and any other connection flags) passed to the `pg_basebackup` command itself — the same host that was used to pull the backup is the host recorded for ongoing streaming replication afterward.

---

**Q5. If a primary is completely destroyed while a standby was stopped and had fallen behind, is there any way to recover the data that was only ever committed on the primary after the standby's last sync?**

No — if that data was never shipped (via WAL) to any standby or archive location before the primary was lost, it's gone. This is exactly the scenario synchronous replication (Task 6) is designed to prevent: by requiring standby confirmation before reporting a commit as successful, a properly configured synchronous setup ensures data isn't considered "durably committed" until it exists in at least one other location.
