# PostgreSQL Physical Files — WAL, Data Files & the Control File

A theory-plus-practice reference tying together PostgreSQL's three physical file types — WAL files, data files, and the control file — with the actual commands and output used to inspect each one.

---
## Table of Contents

- [1. WAL Files — Physical Naming Convention](#1-wal-files--physical-naming-convention)
- [2. Checkpoint Recycling](#2-checkpoint-recycling)
- [3. WAL Files — Logical Naming (LSN)](#3-wal-files--logical-naming-lsn)
- [4. Reading WAL Content with pg_waldump](#4-reading-wal-content-with-pg_waldump)
- [5. "Invalid Record Length" — Not an Error](#5-invalid-record-length--not-an-error)
- [6. Data Files — the Namespace Hierarchy](#6-data-files--the-namespace-hierarchy)
- [7. template0, template1, and postgres](#7-template0-template1-and-postgres)
- [8. Cloning Databases via TEMPLATE](#8-cloning-databases-via-template)
- [9. Mapping Databases to Physical Directories](#9-mapping-databases-to-physical-directories)
- [10. The Control File](#10-the-control-file)
- [11. Shutdown Modes](#11-shutdown-modes)
- [12. Reading Crash Recovery from the Log](#12-reading-crash-recovery-from-the-log)

---

## 1. WAL Files — Physical Naming Convention

```bash
cd /u01/pgsql/18/pg_wal
ls -lrt
```
```
-rw-------. 1 postgres postgres 16777216 Jul 22 22:03 000000010000000000000001
-rw-------. 1 postgres postgres 16777216 Jul 22 22:03 000000010000000000000002
-rw-------. 1 postgres postgres 16777216 Jul 22 22:03 000000010000000000000003
-rw-------. 1 postgres postgres 16777216 Jul 22 22:03 000000010000000000000004
```

**Theory:** every WAL segment file is exactly **16MB** (the default segment size) and is named with a **24-character hexadecimal** string, split into three 8-character blocks:

```
000000010000000000000001
└──┬───┘└──┬───┘└──┬───┘
 timeline  log ID   segment
   ID      (hi bits  number
            of LSN)  within
                     that log
```

| Segment | Meaning |
|---|---|
| `00000001` | **Timeline ID** — increments every time the cluster takes a new timeline branch (after a `pg_rewind`/failover/point-in-time-recovery restore to a specific point, covered in the earlier switchover/failover session) |
| `00000000` | **Log ID** — the high-order 32 bits of the 64-bit Log Sequence Number (LSN); effectively "which 4GB block" of WAL history this segment belongs to |
| `00000001` | **Segment number** — which 16MB segment within that 4GB log ID block this file is (4GB ÷ 16MB = 256 possible segments per log ID, before rolling over to the next log ID) |

**Practical reading:** file `000000010000000000000004` is timeline 1, log ID 0, segment 4 — the fourth 16MB chunk of WAL ever written on this timeline.

---

## 2. Checkpoint Recycling

```bash
psql -c "checkpoint";
```
```
-rw-------. 1 postgres postgres 16777216 Jul 22 22:03 000000010000000000000005
-rw-------. 1 postgres postgres 16777216 Jul 22 22:03 000000010000000000000006
-rw-------. 1 postgres postgres 16777216 Jul 22 22:03 000000010000000000000007
-rw-------. 1 postgres postgres 16777216 Jul 22 22:03 000000010000000000000004
```

**Theory:** rather than deleting old, no-longer-needed WAL segments and creating brand-new empty ones from scratch (both real filesystem operations with real overhead), PostgreSQL **recycles** them — an old segment whose content is no longer required for crash recovery is simply **renamed** to the next needed segment number and its content overwritten going forward. This is precisely why segments `5`, `6`, `7` appear brand new in the listing above while `4` remains — a checkpoint determined segments `1`–`3` were no longer needed and recycled them forward as `5`–`7`; `4` was still required and stayed in place. This is a deliberate efficiency mechanism: reusing existing 16MB files avoids the disk-allocation cost of repeatedly creating fresh ones.

---

## 3. WAL Files — Logical Naming (LSN)

Beyond the physical filename, every individual position *within* the WAL stream has its own address — the **Log Sequence Number (LSN)**.

```
0/080000A8
```

**Theory:** an LSN is written as two hexadecimal numbers separated by a slash: `<high 32 bits>/<low 32 bits>` of the WAL stream's 64-bit byte offset. `0/080000A8` means: log ID `0`, byte offset `080000A8` within it — i.e., roughly "8MB and a bit" into the very first 4GB block of WAL ever generated. This is the addressing scheme used throughout replication monitoring, `pg_current_wal_lsn()`, and recovery target specification — it's a precise byte-level position, not just a segment-level identifier.

---

## 4. Reading WAL Content with pg_waldump

WAL files are stored in **binary format** — not human-readable directly. `pg_waldump` decodes and prints their actual content.

```bash
psql -c "update emp set sal=200 where id=1";
/usr/pgsql-18/bin/pg_waldump 000000010000000000000008
```
```
rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 0/08000028, prev 0/0700A170, desc: RUNNING_XACTS nextXid 770 latestCompletedXid 769 oldestRunningXid 770
rmgr: Heap        len (rec/tot):     72/    72, tx:        770, lsn: 0/08000060, prev 0/08000028, desc: HOT_UPDATE old_xmax: 770, old_off: 2, old_infobits: [], flags: 0x20, new_xmax: 0, new_off: 3, blkref #0: rel 1663/5/16409 blk 0
rmgr: Transaction len (rec/tot):     34/    34, tx:        770, lsn: 0/080000A8, prev 0/08000060, desc: COMMIT 2026-07-22 22:07:48.821377 IST
pg_waldump: error: error in WAL record at 0/80000A8: invalid record length at 0/80000D0: expected at least 24, got 0
```

**Theory — what each field means:**
- **`rmgr`** (resource manager) — which internal PostgreSQL subsystem generated this record. `Standby` records track transaction bookkeeping for replicas; `Heap` records are actual table-data changes; `Transaction` records mark commit/abort boundaries.
- **`lsn`** — this specific record's own address in the WAL stream.
- **`prev`** — the LSN of the *previous* record, forming a linked chain — this is how WAL replay knows the exact sequence to follow during recovery.
- **`tx`** — the transaction ID that generated this record (directly the same `xmin`/transaction ID concept from the ACID/MVCC sessions).
- **`desc`** — a human-readable description of exactly what changed.

**A directly observable detail: `HOT_UPDATE`.** This confirms the row update used PostgreSQL's **Heap-Only Tuple (HOT)** optimization — a fast-path update mechanism available when the updated column isn't part of any index, avoiding the overhead of a full index update alongside the row update.

---

## 5. "Invalid Record Length" — Not an Error

The `pg_waldump` output above ends with what looks like an error:
```
pg_waldump: error: error in WAL record at 0/80000A8: invalid record length at 0/80000D0: expected at least 24, got 0
```

**Theory:** this is **not evidence of corruption** — it's the expected way `pg_waldump` reports reaching the **end of currently-written WAL content**. WAL segment files are pre-allocated at their full 16MB size and padded with zero bytes ahead of where actual data has been written; reading past the last genuine record naturally hits that zero-filled space, which parses as a record with length `0` — hence "invalid record length... got 0." **This message is telling you exactly where the *next* transaction's record will be written.**

**Confirmed directly by the follow-up demonstration:**
```bash
psql -c "update emp set sal=300 where id=1";
/usr/pgsql-18/bin/pg_waldump 000000010000000000000008
```
```
rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 0/080000D0, prev 0/080000A8, desc: RUNNING_XACTS nextXid 771 latestCompletedXid 770 oldestRunningXid 771
rmgr: Heap        len (rec/tot):     72/    72, tx:        771, lsn: 0/08000108, prev 0/080000D0, desc: HOT_UPDATE old_xmax: 771, old_off: 3, old_infobits: [], flags: 0x20, new_xmax: 0, new_off: 5, blkref #0: rel 1663/5/16409 blk 0
rmgr: Transaction len (rec/tot):     34/    34, tx:        771, lsn: 0/08000150, prev 0/08000108, desc: COMMIT 2026-07-22 22:07:54.680526 IST
pg_waldump: error: ... invalid record length at 0/8000178: expected at least 24, got 0
```

The second `UPDATE`'s first record genuinely starts at exactly **`0/080000D0`** — precisely where the first run's "invalid record length" message said the next record would go. **This same mechanism is exactly why a successful crash recovery replay also ends with an "invalid record length" message** (Section 12) — it isn't a recovery failure, it's recovery correctly reaching the natural end of what's actually been written so far.

---

## 6. Data Files — the Namespace Hierarchy

```sql
\du    -- roles
\l     -- databases
\dn    -- schemas
```

**Theory — PostgreSQL's namespace hierarchy, and a direct contrast with Oracle:**

```
Server (cluster / instance)
 └── Database
      └── Schema
           └── Table
```

- **A user** is who connects and is granted privileges — controls access to databases/schemas.
- **A schema** is the logical container that actually holds tables (and other objects like views, sequences, functions).
- **A database** is the logical container that holds one or more schemas.

**The key structural difference from Oracle, worth stating explicitly:** Oracle traditionally maps **one schema to exactly one user** (a schema *is* a user, in practice). PostgreSQL keeps these as genuinely separate concepts — a single database can contain many schemas, and a single user can be granted access across many schemas or databases. This is why PostgreSQL naturally supports multiple applications/teams sharing one database via separate schemas, in a way that doesn't map cleanly onto Oracle's schema=user convention.

---

## 7. template0, template1, and postgres

```
   Name    |  Owner   | Encoding | ...
-----------+----------+----------+
 postgres  | postgres | UTF8     |
 template0 | postgres | UTF8     |
 template1 | postgres | UTF8     |
```

| Database | Purpose |
|---|---|
| `postgres` | The default administrative connection database — where you land by default, and a convenient place to run cluster-wide administrative commands |
| `template0` | An **unmodifiable, pristine, empty** database — the untouched baseline every cluster is initialized with. Never connect to it for real work; its entire purpose is to exist as a guaranteed-clean source to restore from or clone. |
| `template1` | The **default template** used whenever a new database is created without specifying one explicitly — any customization made to `template1` (e.g. installing an extension everyone should have) is automatically inherited by every future `CREATE DATABASE` |

---

## 8. Cloning Databases via TEMPLATE

```sql
-- Clone (implicitly) from template1 — the default behavior of plain CREATE DATABASE
CREATE DATABASE a;

-- Clone explicitly from template0 — guaranteed clean, no template1 customizations included
CREATE DATABASE b TEMPLATE template0;

-- Clone from an existing real database, "a" — copies a's current schema/data as the starting point
CREATE DATABASE c TEMPLATE a;
```

**Theory:** `CREATE DATABASE` is fundamentally a **cloning operation** — every new database starts life as a copy of some existing database's metadata (and, depending on template, its data). Plain `CREATE DATABASE` implicitly uses `template1`; naming a `TEMPLATE` explicitly lets you clone from `template0` (guaranteed pristine) or from **any other existing database**, including one you built and populated yourself — a genuinely useful pattern for quickly spinning up a pre-populated test/staging copy of a real database.

---

## 9. Mapping Databases to Physical Directories

```bash
cd /u01/pgsql/18/base
ls -lrt
```
```
drwx------. 2 postgres postgres 8192 Jul 22 22:02 4
drwx------. 2 postgres postgres    6 Jul 22 22:03 pgsql_tmp
drwx------. 2 postgres postgres 8192 Jul 22 22:08 1
drwx------. 2 postgres postgres 8192 Jul 22 22:08 5
```
```sql
SELECT oid, datname FROM pg_database;
```
```
 oid |  datname
-----+-----------
   5 | postgres
   1 | template1
   4 | template0
```

**Theory:** every database's physical files live in their own directory under `base/`, named after that database's **OID** (object identifier) — not its human-readable name. Cross-referencing the two outputs above: directory `1` is `template1`, `4` is `template0`, `5` is `postgres` — confirming the logical database name you interact with in SQL and the physical directory structure on disk are connected purely through this OID mapping, not the name itself. (The `pgsql_tmp` directory alongside them is the temporary-tablespace working area covered in the earlier background-processes session.)

---

## 10. The Control File

```bash
/usr/pgsql-18/bin/pg_controldata -D /u01/pgsql/18
```
```
pg_control version number:            1800
Catalog version number:               202506291
Database system identifier:           7665391240543557765
Database cluster state:               in production
pg_control last modified:             Wednesday 22 July 2026 10:08:38 PM
Latest checkpoint location:           0/800A240
Latest checkpoint's REDO location:    0/800A1B0
Latest checkpoint's REDO WAL file:    000000010000000000000008
Latest checkpoint's TimeLineID:       1
Latest checkpoint's PrevTimeLineID:   1
Latest checkpoint's full_page_writes: on
Latest checkpoint's NextXID:          0:772
```

**Theory:** the control file (`pg_control`, physically inside the data directory's `global/` subfolder) is the single, small file the postmaster reads on every startup to determine the cluster's current state — this is exactly the file referenced throughout the crash-recovery discussion in the earlier UPDATE-walkthrough session. `pg_controldata` is the tool that reads and prints it in human-readable form (mirroring the earlier certification-quiz answer that `pg_controldata` is the correct tool name, distinct from the file itself, `pg_control`).

**Key fields, explained:**
- **`Database cluster state`** — `in production` means the cluster believes it was shut down cleanly and is safe to serve traffic; other values (e.g. indicating a prior unclean shutdown) are exactly what triggers automatic crash recovery on startup.
- **`Latest checkpoint location`** vs. **`Latest checkpoint's REDO location`** — two related but distinct LSNs: the checkpoint location is where the checkpoint *record itself* was written; the REDO location is the earlier point recovery actually needs to **start replaying from** to reach full consistency (since some writes can be in-flight when a checkpoint begins).
- **`Latest checkpoint's REDO WAL file`** — literally names the physical WAL segment file recovery would begin reading from if a crash happened right now.
- **`TimeLineID`** / **`PrevTimeLineID`** — the current timeline, and what it branched from — relevant after any point-in-time-recovery restore or failover event.
- **`NextXID`** — the next transaction ID to be assigned — directly the same counter tracked throughout the autovacuum/wraparound sessions.

---

## 11. Shutdown Modes

```
smart       quit after all clients have disconnected
fast        quit directly, with proper shutdown (default)
immediate   quit without complete shutdown; will lead to recovery on restart
```

**Theory, connecting each mode to what's already been covered:**
- **`smart`** — the gentlest option: waits indefinitely for every connected client to disconnect on its own before shutting down. Rarely used in practice, since a single idle-but-connected session can block shutdown indefinitely.
- **`fast`** (the default for `pg_ctl stop`) — disconnects all clients immediately, but still performs a **proper, clean shutdown checkpoint** first — this is the normal, safe way to stop PostgreSQL.
- **`immediate`** — the equivalent of `SHUTDOWN ABORT` referenced in the earlier crash-recovery session — skips the clean shutdown checkpoint entirely, guaranteeing that **instance recovery will run on the next startup**, exactly as demonstrated in that session's live `pg_ctl stop -m immediate` demo.

---

## 12. Reading Crash Recovery from the Log

**A successful recovery:**
```
LOG:  redo starts at 0/314C1770
LOG:  invalid record length at 0/38FCEAD8: expected at least 24, got 0
LOG:  redo done at 0/38FCEAA0 system usage: CPU: user: 0.08 s, system: 0.81 s, elapsed: 1.54 s
```

**Theory:** this is a textbook **successful** recovery. `redo starts` names the LSN recovery begins replaying from (matching the control file's REDO location, Section 10); the `invalid record length` line appearing *before* `redo done` is the same natural end-of-written-WAL signal from Section 5 — recovery replayed every genuine record it found and correctly stopped upon reaching the zero-padded, not-yet-written tail of the WAL file. **Seeing this specific message during recovery is a sign everything worked correctly, not a sign of a problem** — it's simply how PostgreSQL recognizes "I have now read everything there is to read."

**An unsuccessful (or at least, differently-terminated) recovery:**
```
LOG:  redo starts at 0/40ADFD18
LOG:  redo done at 0/44FFF180 system usage: CPU: user: 0.05 s, system: 0.37 s, elapsed: 0.47 s
LOG:  checkpoint starting: end-of-recovery immediate wait
LOG:  checkpoint complete: wrote 10969 buffers (66.9%), wrote 3 SLRU buffers; 0 WAL file(s) added,
 redo lsn=0/45000058
LOG:  database system is ready to accept connections
```

**Theory:** notice the **absence** of the "invalid record length" line between `redo starts` and `redo done` here — recovery reached `redo done` without hitting that natural zero-padding boundary the way the successful example did. This is the diagnostic tell that distinguishes the two logs: recovery completed and the instance came up, but not via the same "read everything, hit the expected end" path as the clean example above — worth specifically investigating (checking for any WARNING/ERROR lines immediately surrounding this section, and confirming the archive/WAL source was actually complete) rather than assuming it's equivalent to the first case just because the instance did eventually report "ready to accept connections."

**Also worth noting, from the second log:** the **`end-of-recovery immediate` checkpoint** — PostgreSQL always performs a checkpoint immediately upon completing recovery, regardless of which path got it there, to establish a fresh, known-good baseline before accepting any new connections.

```bash
/usr/pgsql-18/bin/pg_controldata -D /u01/pgsql/data | grep location
```
```
Latest checkpoint location:           0/40ADFD08
Latest checkpoint's REDO location:    0/40ADFD18
```

**Practical technique confirmed:** piping `pg_controldata` through `grep` is a quick way to check just the checkpoint/REDO location fields directly, without scrolling through the full output — useful for a fast sanity check of exactly where the cluster's last known-good recovery point currently sits.
