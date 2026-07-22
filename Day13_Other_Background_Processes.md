# Background Writer, Checkpointer & Autovacuum Launcher — Session Notes

This session picks up PostgreSQL's background-process architecture and works through three processes in detail: the background writer, the checkpointer, and the autovacuum launcher — what each one does, how they interact, and the first parameter-level view of autovacuum's cleanup logic. The session closes with a live demo of `pg_stat_all_tables` and the autovacuum trigger formula.

---
## Table of Contents

- [1. Recap: Memory, Background Processes, Physical Files](#1-recap-memory-background-processes-physical-files)
- [2. Background Writer](#2-background-writer)
- [3. Checkpointer](#3-checkpointer)
- [4. Background Writer vs Checkpointer — The Health Signal](#4-background-writer-vs-checkpointer--the-health-signal)

---

## 1. Recap: Memory, Background Processes, Physical Files

Quick recap of the architecture discussed in an earlier session, using the update-statement walkthrough as the anchor:

1. `UPDATE` copies the current row within the table first (to satisfy isolation — readers and writers shouldn't interfere with each other).
2. The row is brought into memory (shared buffers) and modified there.
3. On `COMMIT`, the change is written to the **WAL file** first — the "commit complete" message on screen means the change is durable in WAL, **not** that it's in the data file yet.
4. Sometime later, a background process copies that change from memory to the **data file** on disk.

```
1,1,100  →  UPDATE  →  copy in shared buffers, modify to 1,1,200
                              ↓ COMMIT
                        written to WAL file
                              ↓ (later)
                        written to data file
```

---

## 2. Background Writer

**Job:** continuously flush dirty (modified) buffers from shared buffers to the data files, so the amount of unwritten changed data sitting in memory stays low.

**Loop, 24×7:**
1. Wake up.
2. Check shared buffers for dirty pages.
3. If found, write up to `bgwriter_lru_maxpages` pages to disk (default: 100).
4. Sleep for `bgwriter_delay` (default: 200ms).
5. Repeat.

**Worked example:** 150 dirty buffers accumulated. Background writer wakes up, writes 100, sleeps 200ms, wakes up, writes the remaining 50.

**A dirty buffer** is simply a buffer that has been modified in memory since it was read from disk. A plain `SELECT` never creates a dirty buffer; an `UPDATE`, `INSERT`, or `DELETE` does.

**Tuning for a background writer that's falling behind** (dirty buffers consistently piling up in shared buffers, causing performance problems):

```sql
-- More aggressive: write more pages per wake cycle, sleep less between cycles
bgwriter_lru_maxpages = 1000
bgwriter_delay = 10
```

Making `bgwriter_lru_maxpages` bigger and `bgwriter_delay` smaller makes the background writer work closer to continuously instead of in small bursts with long naps.

---

## 3. Checkpointer

**Job:** not to write dirty buffers routinely (that's the background writer's job) — the checkpointer's job is to **guarantee** that, at a known point in time, all dirty buffers as of that point have definitely been written to disk, and to record that guarantee in the control file.

- Invoked on a timer: `checkpoint_timeout` (default: **5 minutes**).
- When invoked, it writes out *any dirty buffers still remaining* in shared buffers at that moment (not routine writing — a catch-all sweep).
- It then updates the **control file** with the checkpoint location — this is the starting point crash/instance recovery replays WAL from.

**Health signal:** if the checkpointer is regularly finding and writing a lot of leftover dirty buffers, that means the background writer isn't keeping up. In a healthy system, the number of buffers written by the background writer should consistently be **higher** than the number written by the checkpointer. If it's the other way around, that's the signal to make the background writer more aggressive (Section 2) — otherwise, leave both at their defaults.

**Why not just increase `checkpoint_timeout` to reduce checkpoint frequency?** Because it directly increases recovery time. The control file only knows about the *last* checkpoint's position — a longer interval between checkpoints means more WAL has to be replayed during crash recovery before the database is consistent again. Keep `checkpoint_timeout` at 5 minutes or lower; don't raise it to 10–15 minutes.

---

## 4. Background Writer vs Checkpointer — The Health Signal

| Background writer writing more than checkpointer | Background writer writing less than checkpointer |
|---|---|
| Healthy — background writer is keeping shared buffers clean on its own | Background writer isn't keeping up; checkpointer is doing cleanup work it shouldn't need to do |
| Leave `bgwriter_delay`/`bgwriter_lru_maxpages` at defaults | Make background writer more aggressive (lower delay, higher max pages) |

---

## 5. Asynchronous I/O (PostgreSQL 18)

New in PostgreSQL 18: instead of fetching one data page at a time from disk, contiguous pages can be fetched together in a single I/O operation, reducing the number of round trips for scans that touch a lot of pages.

**When it helps:** `SELECT` without a `WHERE` clause (or any scan that reads many contiguous pages), `SELECT COUNT(*)`, and `VACUUM`.

**When it doesn't apply:** a `SELECT` with a `WHERE` clause that goes through an index and fetches scattered individual rows falls back to the traditional one-page-at-a-time method regardless of this setting.

**Controlled by `io_method`:**

```sql
-- postgresql.conf
io_method = worker     -- default; recommended, leave as-is
```

| Value | Meaning |
|---|---|
| `worker` | **Default.** Enables async I/O using PostgreSQL's own background worker processes. Recommended — leave this as-is. |
| `sync` | Disables async I/O — falls back to the traditional PostgreSQL 17-and-earlier one-page-at-a-time method. |
| `io_uring` | Enables async I/O using the operating system's `io_uring` facility (Linux kernel-level async I/O) instead of PostgreSQL's own workers. |

**Guidance given:** stick with the default (`worker`). Switching to `io_uring` requires OS-level kernel support and sign-off from the systems/OS team (most environments won't easily grant this), and the measured performance difference between `worker` and `io_uring` is typically only around 5–10% — and negligible on some operating systems. Don't disable async I/O (`sync`) either, since it's a net performance improvement with the default setting. In short: leave `io_method` at its default and move on.

**To actually measure the difference**, if curious: create a several-GB table, run `SELECT * FROM table_name` (no `WHERE`) with the setting enabled vs. disabled, and compare timings.

---

## Local Memory Recap and Temporary Tables

Quick recap of local (per-session) memory regions:

| Region | Used for |
|---|---|
| `work_mem` | Sort/hash operations, e.g. `ORDER BY` — bigger `work_mem` means faster sorts |
| `maintenance_work_mem` | Vacuum and other maintenance operations — bigger means faster vacuum |
| Temp buffers | Temporary tables created within a session |

**Demo — temporary tables live only for the session:**

```sql
CREATE TEMPORARY TABLE test_table (id int);
INSERT INTO test_table VALUES (1, 100);
SELECT * FROM test_table;    -- works fine

-- \q, then reconnect (new session)
SELECT * FROM test_table;    -- ERROR: relation "test_table" does not exist
```

The temp table lived in the session's temp buffers; once the session ended, it — and its data — disappeared entirely.

**Where a temp table actually lives:** the data sits in memory (temp buffers) up to a configured size; anything that doesn't fit spills to disk, into the cluster's temporary tablespace. Either way, all of it — in-memory and on-disk — is removed once the session that created it terminates.
