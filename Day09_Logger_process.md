# INSERT/DELETE Walkthroughs, the Logger Process & Database-Per-Instance Guidance — Session Notes

Completing the CRUD architecture walkthrough with `INSERT` and `DELETE` (both reusing the `UPDATE` mechanics from the prior session), an important correction on how the background processes actually coordinate ("nobody informs nobody"), then a full dive into the logger process — where/when/what to log — and closing with practical guidance on how many databases belong in one PostgreSQL instance and why physical backups are always instance-wide, never per-database.

---
## Table of Contents

- [1. Correction: Nobody Informs Nobody](#1-correction-nobody-informs-nobody)
- [2. The INSERT Walkthrough](#2-the-insert-walkthrough)
- [3. The DELETE Walkthrough](#3-the-delete-walkthrough)
- [4. UPDATE Is Really DELETE + INSERT](#4-update-is-really-delete--insert)
- [5. The Logger Process — Three Subsections](#5-the-logger-process--three-subsections)
- [6. Where to Log](#6-where-to-log)
- [7. When to Log — the Message-Level Hierarchy](#7-when-to-log--the-message-level-hierarchy)
- [8. What to Log — log_statement and log_min_duration_statement](#8-what-to-log--log_statement-and-log_min_duration_statement)
- [9. Log File Naming and Rotation — a Real Gotcha](#9-log-file-naming-and-rotation--a-real-gotcha)
- [10. Log Rotation by Size — Don't Touch It](#10-log-rotation-by-size--dont-touch-it)
- [11. How Many Databases Belong in One Instance](#11-how-many-databases-belong-in-one-instance)
- [12. Physical Backups Are Always Instance-Wide](#12-physical-backups-are-always-instance-wide)
- [13. Questions Discussed in This Session](#13-questions-discussed-in-this-session)

---

## 1. Correction: Nobody Informs Nobody

**An important clarification of the previous session's simplified explanation:** in the `UPDATE` walkthrough, the WAL writer was described as "informing" the checkpointer, which in turn "informs" the background writer. **Explicitly corrected here:** this was a deliberate simplification for teaching purposes only — in reality, **no background process directly messages another.** Each process does its own job independently and silently, on its own schedule:
- The **WAL writer's** job: move data from `wal_buffers` to the WAL file.
- The **background writer's** job: move data from `shared_buffers` to the data file.
- The **checkpointer's** job: periodically (every 5 minutes by default) confirm that all dirty (modified) buffers have actually been flushed to disk, and trigger the background writer if any remain.

**The point of the earlier simplified version** was purely to establish the correct *order* — which process's work logically depends on which other's — not to imply literal inter-process signaling.

---

## 2. The INSERT Walkthrough

```sql
INSERT INTO employee VALUES (2, 200);
```

**Key distinction from `UPDATE`, stated directly: the only difference is where the data comes from.**
- For `UPDATE`, the source is the **existing row on disk** — it has to first be copied and brought into memory.
- For `INSERT`, the source is the **client** — brand new data being sent in, with nothing on disk to copy from first.

**Steps, otherwise identical to `UPDATE` from this point on:**
1. New row data arrives from the client directly into `shared_buffers`.
2. Copied into `wal_buffers` → client sees **"1 row inserted."**
3. On `COMMIT`, the WAL writer flushes `wal_buffers` to the WAL file → client sees **"commit complete."**
4. At some later point, the background writer flushes the new row from `shared_buffers` to the data file; the checkpointer periodically confirms this has happened.

**Analogy used to make the client → memory → disk flow concrete:** typing into Notepad — the text exists only in memory while you're typing (not yet on disk); pressing "Save" is the equivalent of `COMMIT`, moving it to disk.

---

## 3. The DELETE Walkthrough

```sql
BEGIN;
DELETE FROM employee WHERE id = 1;
```

**Steps:**
1. The target row is **marked as deleted** in place (not physically removed).
2. Brought into memory (`shared_buffers`), copied to `wal_buffers`.
3. Client sees **"1 row deleted."**
4. On `COMMIT`, the WAL writer flushes to the WAL file → client sees **"commit complete"** — and only *at this point* does the row's "marked as deleted" status become genuinely final (dead).

**Confirmed directly: neither the background writer nor the checkpointer has anything to do for a `DELETE`** in the same way they do for `UPDATE`/`INSERT` — there's no new data-file content to write, since a delete doesn't add or overwrite a value, only marks existing content as no longer valid.

**Who actually does the physical cleanup:** the **autovacuum launcher** — running continuously in the background (`ps -ef | grep postgres` shows it as a distinct, always-present process) — periodically checks for dead tuples across all databases and reclaims that space. **Confirmed live via the alert log:** the autovacuum launcher's "processing database postgres" message appeared at roughly one-minute intervals in this lab environment — it checks on a schedule regardless of whether there's actually anything to clean up; if nothing needs cleaning, it simply goes back to sleep.

---

## 4. UPDATE Is Really DELETE + INSERT

**Stated directly, tying Sections 2 and 3 together:** internally, `UPDATE` in PostgreSQL is architecturally equivalent to a `DELETE` of the old row version plus an `INSERT` of the new one — the old version is marked dead (exactly like a `DELETE`), and the new version is written as a fresh row (exactly like an `INSERT`). This is the same MVCC row-versioning mechanism from the ACID properties session, now connected directly to the `INSERT`/`DELETE` mechanics covered today.

---

## 5. The Logger Process — Three Subsections

```bash
ps -ef | grep postgres
```
The **logger process** is one of PostgreSQL's background processes — its job is to write event information to the alert log. Its configuration in `postgresql.conf` is organized into three conceptual questions:

1. **Where to log** — where should this information physically be written?
2. **When to log** — how much detail/verbosity should be captured?
3. **What to log** — which specific kinds of events should be captured?

---

## 6. Where to Log

```ini
log_destination = 'stderr'
logging_collector = on
log_directory = 'log'          # relative to the data directory by default
log_filename = 'postgresql-%a.log'
```

**Production recommendation, stated directly:** move `log_directory` **out of the data directory** — e.g. to a dedicated mount point like `/u01/logs` — so log growth doesn't compete for space with the actual database files, and so log volume is easy to monitor/manage independently.

```bash
mkdir -p /u01/logs
```
```ini
log_directory = '/u01/logs'
```

---

## 7. When to Log — the Message-Level Hierarchy

```ini
log_min_messages = warning   # default
```

**The severity hierarchy, from most to least severe:** `panic` → `fatal` → `log` → `error` → `warning` → `notice` → `info` → `debug1` → `debug2` (increasingly verbose going down). **Setting a level captures that level and everything *more* severe than it** — e.g. `warning` captures `warning`, `error`, `log`, `fatal`, and `panic`, but not `notice`/`info`/`debug`.

**Live demonstration:** with `log_min_messages = warning`, a normal stop/start/checkpoint cycle produced only a handful of log lines (~7 for stop, 2 for a checkpoint). Switching to `debug2` and repeating the exact same operations produced dramatically more output — the same stop/start/checkpoint sequence, but now showing extensive internal detail (down to the specific interval and internal reasoning behind autovacuum launcher checks, WAL position details, etc.).

**Explicit, repeated production guidance:** **never set anything more verbose than `warning` in a production system.** More verbose levels flood the alert log with detail that's never actually used for real troubleshooting and just adds noise and disk usage. **`debug1`/`debug2` are only appropriate in a test/learning environment**, specifically when deliberately trying to understand PostgreSQL's internal behavior for a specific activity — turn it on, observe, then turn it back off; never leave it on as a standing configuration.

---

## 8. What to Log — log_statement and log_min_duration_statement

```bash
cat postgresql.conf | grep log_
```

**`log_statement`** — controls whether SQL statements themselves get logged, and which kinds:
```ini
log_statement = 'none'   # default; other options: 'ddl', 'mod', 'all'
```
Setting `all` logs **every** statement executed, regardless of type — used for full auditing, at the cost of substantial log volume.

**`log_min_duration_statement`** — logs any statement whose execution time exceeds a given threshold:
```ini
log_min_duration_statement = 2000   # milliseconds — log any statement taking > 2 seconds
```
**Purpose, stated directly:** this is specifically for identifying **long-running queries** after the fact — reviewing the alert log later (e.g. at the end of the day) to see which statements crossed the threshold, as a starting point for performance tuning/troubleshooting, rather than needing to catch slow queries live.

**Confirmed directly: none of these logging-related parameters require a restart** — all take effect via `pg_ctl reload` (or `SELECT pg_reload_conf();`), consistent with earlier sessions' guidance that logging configuration is always a reload-only change.

---

## 9. Log File Naming and Rotation — a Real Gotcha

```ini
log_filename = 'postgresql-%a.log'   # default — %a = day of week (Mon, Tue, Wed...)
```

**A genuine, easy-to-miss production issue:** with the default `%a` pattern, the alert log file is named by **day of the week**, not a full date — which means **after 7 days, the file gets silently overwritten**, and the previous week's same-weekday log content is permanently lost.

**The fix — change this on day one of any real deployment:**
```ini
log_filename = 'postgresql-%Y%m%d.log'   # year-month-day — never collides/overwrites
```

**Explicit prioritization given between the two "where to log" settings:** changing `log_directory` (location) is optional/situational — leaving it inside the data directory is technically fine. **Changing `log_filename` away from the day-of-week default is not optional** — it should be done immediately on any real system, since the data-loss consequence of leaving it as-is is real and easy to hit without noticing.

**Sizing consideration for retention planning:** if daily log volume is, say, 10MB and retention is 10 days, that's roughly 100MB of accumulated log data — manageable. But if daily volume reaches something like 1GB (a heavily logged environment), the same 10-day retention becomes ~100GB — a genuinely significant amount that argues for moving logs to dedicated storage.

**Manual cleanup, since PostgreSQL has no built-in retention/purge mechanism:**
```bash
find /u01/logs -name "*.log" -mtime +90 -delete
```
Scheduled via `cron`, this retains roughly the most recent 90 days and purges older files — the practical, standard way logs are managed in real deployments, based on whatever retention SLA applies.

---

## 10. Log Rotation by Size — Don't Touch It

**Explicitly, directly discouraged:** PostgreSQL exposes parameters that appear to offer size-based and time-based log rotation (e.g. rotate after reaching some size like 500MB, independent of the daily filename change). **Direct guidance: don't use these.** They were described as **not working reliably as documented/intended**, and using them tends to create confusion and a real risk of losing log content unexpectedly, rather than providing the clean rotation behavior their names suggest.

**Also clarified: PostgreSQL's rotation, where it does work, does not "archive" the old file the way Oracle-style rotation might** — when a size-based rotation actually triggers, the old content is effectively **replaced**, not kept as a separate retained copy the way you might expect from a traditional log-rotation tool.

**A rough workaround mentioned, if intra-day size-based splitting is genuinely needed:** using OS-level tools (e.g. scripted `dd`/copy tricks keyed to date/time) to manually split files — described as a "hack," not a supported or recommended standard practice.

---

## 11. How Many Databases Belong in One Instance

**Direct guidance given, framed as a practical rule of thumb rather than a hard technical limit:**
- **5 databases per instance is a good number.**
- **5–10 is acceptable.**
- **More than 10 warrants rethinking your architecture** — regardless of how small the individual databases are.

**Worked example — a bank's core systems:** major, business-critical lines of business (e.g. net banking and securities trading) should each get their **own dedicated instance**, not merely separate databases sharing one instance. Smaller, less critical services can be grouped together in a shared instance's multiple databases, but the truly critical applications should be isolated at the instance level.

**Why the number matters, beyond just organizational tidiness — three concrete reasons given:**
1. **Backup granularity** (see Section 12) — since all databases in one instance share one physical backup, more databases means more blast radius per backup/restore operation.
2. **Shared resource contention** — every database in an instance shares the same CPU and memory; connection counts add up across all of them (e.g. two applications at 100 connections each is already 500 total against the instance's `max_connections`), directly driving compute sizing decisions.
3. **Blast radius of instance loss** — if the instance goes down, every database inside it goes down together; concentrating too many unrelated applications in one instance means a single failure event affects all of them simultaneously.

---

## 12. Physical Backups Are Always Instance-Wide

**A direct, important architectural point, raised via comparison to SQL Server:** in SQL Server, a single database within a multi-database instance can be backed up **standalone** — but **PostgreSQL has no equivalent capability**. A PostgreSQL physical backup always captures **the entire instance's physical files** — every database within it — together, as one atomic unit. There is no way to physically back up (or restore) just one database out of a multi-database cluster.

**Directly confirmed as a genuine architectural constraint, not a workaround-able limitation:** you cannot schedule "database A backed up today, database B backed up tomorrow" — a physical backup operation always includes everything in the instance, every time it runs.

**This is exactly why the database-count guidance in Section 11 matters in practice:** more databases sharing one instance means every backup and restore operation necessarily involves all of them together — directly reinforcing why business-critical applications are better isolated into their own dedicated instances rather than sharing one with unrelated services.

---

## 13. Questions Discussed in This Session

**Q1. Does an `INSERT` write directly to the data file, or does it also pass through memory first?**

Through memory first, exactly like `UPDATE` — the only difference is the *source* of the data (client, rather than an existing on-disk row). New data always lands in `shared_buffers` and `wal_buffers` before ever reaching a physical file, following the same commit-then-eventually-flush sequence as any other write.

---

**Q2. Is the autovacuum launcher only triggered reactively, right after a delete happens?**

No — it runs continuously, checking on its own periodic schedule (observed at roughly one-minute intervals in this session's demo) regardless of whether there's actually anything to clean up. If it finds no dead tuples needing attention, it simply goes back to sleep until its next scheduled check; it's not event-triggered by a specific `DELETE` statement.

---

**Q3. Do changes to logging-related parameters (`log_min_messages`, `log_statement`, `log_min_duration_statement`, etc.) require a database restart?**

No — confirmed directly and repeatedly: all logging-related configuration changes take effect via a simple reload (`pg_ctl reload` or `SELECT pg_reload_conf();`), never requiring a full restart.

---

**Q4. If SQL Server and Oracle allow standalone single-database backups within a multi-database instance, why doesn't PostgreSQL?**

This is a genuine architectural difference, not a missing feature to work around — PostgreSQL's physical backup mechanism operates on the instance's entire physical file set as one unit; there's no per-database boundary at the physical-file level the way there is in SQL Server. This is precisely why the course's guidance is to keep truly critical, independent applications in their own separate instances rather than relying on being able to back them up independently within a shared one.
