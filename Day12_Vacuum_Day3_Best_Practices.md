# Autovacuum Best Practices, VACUUM Internals & MVCC — Session Notes

This session moves from autovacuum's mechanics (previous session) into best practices: scheduling manual vacuum alongside autovacuum, transaction ID wraparound monitoring, `VACUUM FULL` vs `pg_repack`, and a hands-on demo of MVCC row versioning using `xmin`/`xmax` and the `pageinspect` extension. It closes with a full reference table of `VACUUM` command variants.

---
## Table of Contents

- [1. Manual Vacuum as a Best Practice](#1-manual-vacuum-as-a-best-practice)
- [2. Why Repeated Manual Vacuums Get Faster](#2-why-repeated-manual-vacuums-get-faster)
- [3. Disabling Autovacuum on Specific Tables](#3-disabling-autovacuum-on-specific-tables)
- [4. Transaction ID Wraparound Monitoring](#4-transaction-id-wraparound-monitoring)
- [5. Speeding Up a Slow Vacuum](#5-speeding-up-a-slow-vacuum)
- [6. VACUUM FULL vs pg_repack](#6-vacuum-full-vs-pg_repack)
- [7. MVCC Row Versioning Demo](#7-mvcc-row-versioning-demo)
- [8. Freezing Explained](#8-freezing-explained)
- [9. VACUUM Command Variants](#9-vacuum-command-variants)
- [10. Vacuum and Index Bloat](#10-vacuum-and-index-bloat)
- [11. Questions Discussed in This Session](#11-questions-discussed-in-this-session)

---

## 1. Manual Vacuum as a Best Practice

Autovacuum runs 24×7 by design, but it doesn't know or care about your business hours — it might pick up a business-critical table for cleanup right in the middle of peak traffic, slowing down (not locking, but slowing) selects/inserts/updates on that table.

**Best practice:** schedule a manual `VACUUM` on all — or at minimum, your business-critical — tables during a low-traffic window, before business hours start.

```sql
-- example: scheduled at 3 AM, business hours start at 9 AM
VACUUM;   -- or VACUUM <table_name> per table, scripted for all production tables
```

By the time autovacuum's launcher scans the tables during business hours, it finds nothing over threshold and simply goes back to sleep for `autovacuum_naptime` — no interference with your peak-hours traffic. This doesn't replace autovacuum (autovacuum keeps running regardless) — it just means autovacuum rarely finds anything left to do during the hours that matter most.

There's no downside to running a manual vacuum daily: it's not a full table rescan every time (see Section 2), and there's no conflict between a scheduled manual vacuum and autovacuum — if one is already vacuuming a table, the other simply waits for it to finish rather than starting a duplicate pass on the same table.

---

## 2. Why Repeated Manual Vacuums Get Faster

Every vacuum pass updates two files that track what's already been scanned: the **visibility map** and the **free space map**. On the next vacuum pass, PostgreSQL uses these to skip pages that haven't been touched since the last vacuum, instead of rescanning the entire table.

**Worked example — 10GB table:**

| Vacuum pass | Duration | Why |
|---|---|---|
| 1st | ~2 minutes | Full table scan — nothing has been vacuumed before |
| 2nd | ~20–30 seconds | Only scans pages modified since pass 1 |
| 3rd (daily) | ~2–3 seconds | Only scans pages modified in the last 24 hours |

Scheduling a manual vacuum every day doesn't mean paying the "first pass" cost every day — each subsequent run only has to account for that day's changes.

---

## 3. Disabling Autovacuum on Specific Tables

```sql
ALTER TABLE table_name SET (autovacuum_enabled = false);
```

Or, less drastically, make a specific table's autovacuum trigger less sensitive by raising its scale factor or base threshold at the table level:

```sql
ALTER TABLE table_name SET (autovacuum_vacuum_scale_factor = 0.4);   -- 40% instead of default 20%
ALTER TABLE table_name SET (autovacuum_vacuum_threshold = 5000);      -- instead of default 50
```

**Guidance given:** avoid per-table tuning of these numbers where possible — it adds unnecessary complexity to reason about later. The simpler and preferred approach is scheduling a manual vacuum during off-hours (Section 1) rather than disabling or detuning autovacuum per table.

**If you do disable autovacuum on a table, a scheduled manual vacuum on that table becomes mandatory**, not optional — otherwise the table will bloat indefinitely with nothing cleaning it up.

---

## 4. Transaction ID Wraparound Monitoring

**The concept:** PostgreSQL transaction IDs are a finite 32-bit counter. As they approach exhaustion (2 billion), old transaction IDs must be "frozen" (marked as permanently in the past) to make room — this is what autovacuum's freeze process does. If freezing falls too far behind transaction generation, PostgreSQL forces the issue by going into a read-only "emergency" mode to protect data integrity.

**Two percentages worth tracking, both derived from a monitoring query against `age(datfrozenxid)`-style catalog data:**

| Metric | Relative to | Meaning at 100% |
|---|---|---|
| Percentage towards emergency autovacuum | `autovacuum_freeze_max_age` (default 200 million) | Autovacuum will force an aggressive freeze pass regardless of other settings |
| Percentage towards wraparound | The hard 2 billion transaction ID limit | Database is forced into read-only mode to prevent data corruption |

**Operational guidance given:**
- Don't wait until 99% to act — start investigating and taking action once utilization crosses **70%**. Below 30–40%, day-to-day fluctuation (the occasional long transaction) isn't a concern.
- **How much runway you actually have depends entirely on your growth rate**, not the percentage alone: a system generating 1% transaction ID utilization per day still has roughly 90 days of runway even at 10–12%. A system generating 10% per day has roughly 9–10 days of runway at the same percentage. Always look at rate of growth, not just the current number.

**Fast-tracking a vacuum during an emergency (near 90%+ utilization):**

```sql
VACUUM (INDEX_CLEANUP OFF) table_name;
```

Skipping index cleanup makes the vacuum pass faster because freezing transaction IDs — the urgent problem — is not a performance issue in itself; it's specifically about avoiding wraparound. Follow up with a manual `REINDEX` afterward once the emergency has passed (see Section 10 for why skipping index cleanup has a cost).

---

## 5. Speeding Up a Slow Vacuum

You cannot make an in-progress vacuum run faster — there's no "turbo" signal to send it. If a vacuum has been running for 30+ minutes and needs to go faster:

1. Terminate it: `SELECT pg_terminate_backend(pid);` (using the PID of the vacuum's backend/worker process).
2. Re-run vacuum with more aggressive settings.

**Settings that make a re-run faster:**
- Remove the throttling delay so the vacuum worker doesn't sleep between batches (`vacuum_cost_delay` — default 2ms for autovacuum workers, can be set to 0 for a manual vacuum you want to run flat-out).
- Increase `maintenance_work_mem` for the session — bigger memory for the vacuum's internal work translates to faster completion. Guidance given: roughly 20% of `shared_buffers`, or in the 1–2GB range — don't allocate the whole machine's RAM to it.

**Past vacuum durations are not reliable predictors of future ones** — the first vacuum on a table is always the slowest (full scan); every subsequent vacuum only accounts for the delta since the last run (Section 2), so historical timing isn't directly comparable run-to-run unless the workload pattern is stable.

**Finding which tables are slow to vacuum in practice:** `pgBadger` reports (parsed from PostgreSQL logs) surface which tables are consistently taking longest to vacuum — use that list to prioritize manual vacuum scheduling on the worst offenders, or simply order tables by size and target the largest ones first, since larger tables are more likely to be slow.

---

## 6. VACUUM FULL vs pg_repack

**Plain `VACUUM`** marks dead tuple space as reusable *within the table* for future inserts — it does **not** return that space to the operating system.

**`VACUUM FULL`** rewrites the entire table into a new file and swaps it in, which does return the reclaimed space to the OS. Conceptually:

```sql
BEGIN;
  -- exclusive lock on the table for the duration
  CREATE TABLE test_new AS SELECT * FROM test;
  DROP TABLE test;
  ALTER TABLE test_new RENAME TO test;
COMMIT;
```

Because it holds an exclusive lock on the table for the entire rebuild, `VACUUM FULL` is normally scheduled as a maintenance-window activity — connections terminated, external connectivity disabled, run during a planned downtime.

**`pg_repack`** (a widely used extension) achieves the same end result — a compacted table with space returned to the OS — without holding an exclusive lock for the whole operation:

```sql
BEGIN;
  -- track ongoing changes (via a log/trigger), not a table lock
  -- create new table, copy existing data
  -- apply tracked changes to the new table
  -- brief exclusive lock: swap/rename tables
  -- drop old table
COMMIT;
```

The only real downtime is the brief exclusive lock during the final rename/swap — everything else (copying data, catching up on concurrent changes) happens without blocking the table. `pg_repack --verbose` shows the internal steps as they happen.

**When to reach for either:** neither is a routine, always-on tool — use them when a table is repeatedly getting bloated and staying bloated despite regular vacuuming (e.g. heavy update/delete churn outpacing space reuse).

---

## 7. MVCC Row Versioning Demo

Using the `pageinspect` extension to look directly at physical row versions:

```sql
CREATE TABLE vt (id int, salary int);
INSERT INTO vt VALUES (1, 100);
INSERT INTO vt VALUES (2, 300);

SELECT * FROM vt;
SELECT xmin, xmax, * FROM vt;
```

`xmin` is the transaction ID that inserted the row version; `xmax` is the transaction ID that deleted/superseded it (`0` means still current).

```sql
UPDATE vt SET salary = 300 WHERE id = 1;
```

An `UPDATE` in PostgreSQL is implemented as **insert a new row version + mark the old row version dead** — it is not an in-place modification.

```
Before:  [xmin=1912, xmax=0]    1, 100    <- current
After:   [xmin=1912, xmax=1914] 1, 100    <- dead, but still occupying space
         [xmin=1914, xmax=0]    1, 200    <- new current version
```

**Any session** doing `SELECT * FROM vt WHERE id = 1` will first land on the row at the original position (`xmin=1912`), see that it has an `xmax` (meaning it's superseded), and follow the chain to the current version (`xmin=1914`) — for **every session**, not just the one that ran the update, since this is a committed transaction. This is a real performance cost: every read potentially has to walk through one or more dead versions to reach the live one, and the dead version's disk space is not reclaimed until `VACUUM` runs.

```sql
VACUUM vt;
```

After `VACUUM`: the dead row version's slot is freed for reuse by **future inserts on that same table** (not any other table, and not immediately visible as literally "empty" — it's marked reusable). The visibility map is updated so subsequent selects go straight to the live version.

```sql
VACUUM FULL vt;
```

After `VACUUM FULL`: the space is compacted out of the table file entirely and given back to the OS.

---

## 8. Freezing Explained

Freezing and vacuuming are related but distinct: plain `VACUUM` removes dead tuples; freezing specifically deals with transaction ID exhaustion by marking old-enough row versions as permanently visible, regardless of transaction ID, so their original `xmin` no longer needs to be tracked for visibility purposes.

Historically (PostgreSQL versions before 9.4), freezing was implemented by literally replacing a row's `xmin` with the special `FrozenTransactionId` value (`2`). From 9.5 onward, this is handled through internal tuple-header flags instead — you will not see `xmin` visibly change to `2` in newer versions, but the row is still frozen (its visibility no longer depends on tracking a specific transaction ID).

```sql
VACUUM (FREEZE) vt;
```

---

## 9. VACUUM Command Variants

| Command | What it does |
|---|---|
| `VACUUM employee;` | Removes dead tuples, marking their space reusable **within the table** — does **not** release space to the OS (only `VACUUM FULL` does that) |
| `ANALYZE employee;` | Gathers statistics for the optimizer — does not touch dead tuples |
| `VACUUM ANALYZE employee;` | Both — vacuum, then gather statistics |
| `VACUUM (FREEZE) employee;` | Vacuum plus aggressively freezing eligible transaction IDs |
| `VACUUM (INDEX_CLEANUP OFF) employee;` | Vacuum the table's dead tuples, but skip cleaning up the table's indexes (see Section 10 for the tradeoff) |
| `VACUUM (FREEZE, INDEX_CLEANUP OFF) employee;` | Combination — used for emergency wraparound situations where speed matters more than index cleanliness (Section 4) |
| `VACUUM;` (no table name) | Vacuums every table in the current database |
| `vacuumdb --all` | Vacuums every database in the cluster, one table at a time, cluster-wide |
| `VACUUM FULL employee;` | Rewrites the table, releasing space back to the OS (Section 6) — can be cancelled at any point mid-run |

---

## 10. Vacuum and Index Bloat

Vacuuming a table doesn't just clean the table's own dead tuples — it normally also cleans up the corresponding entries in that table's **indexes**. When a row is updated, the old index entry still points at the now-dead row version until vacuum cleans it up.

```
Index entry: employee_id=2 → points to old row location
UPDATE employee SET salary = 200 WHERE id = 2;   -- new row version created elsewhere
```

Immediately after the update, the index still points at the old (now-dead) location. Once `VACUUM` runs normally, it updates the index entry to point at the new location. **But if you run with `INDEX_CLEANUP OFF`** (used for emergency wraparound situations, Section 4), the index is left untouched — it still points at now-nonexistent locations, which effectively makes the index useless (every lookup through it falls back to scanning). This is an acceptable, deliberate tradeoff during an active wraparound emergency, because the immediate priority is freezing transaction IDs, not performance — but it must be followed up with a manual `REINDEX` once the emergency is resolved.

---

## 11. Questions Discussed in This Session

**Q1. Is `pg_repack` truly online, or does it still involve some downtime?**

There's a small amount of downtime — specifically during the final rename/swap step, which requires a brief exclusive lock. Everything before that (copying the table's data, catching up on concurrent changes tracked during the copy) happens without locking the table. This is why `pg_repack` is preferred over `VACUUM FULL` when minimizing downtime matters — the exclusive-lock window is much shorter.

---

**Q2. What does "nap time" mean for autovacuum?**

It's the sleep interval (`autovacuum_naptime`, default 1 minute) the autovacuum launcher rests for after finishing a full scan of all tables, before scanning again.

---

**Q3. Is there a defined way to set up alerting on transaction ID utilization?**

Not something PostgreSQL provides out of the box as an alert — the practical approach is: (1) have a custom query that computes current transaction ID utilization percentage (as used in Section 4's demo), and (2) integrate that query into whatever monitoring tool the project already uses (CloudWatch or otherwise), so it's tracked and alertable the same way any other custom metric would be. The alert log will also start surfacing warning messages as utilization crosses internal thresholds, but a proactive dashboard/alert built on the custom query is the recommended approach for catching it early.

---

**Q4. Is there any overlap risk between a scheduled manual vacuum and autovacuum running on the same table at the same time?**

No. If autovacuum is already vacuuming a table, a manual vacuum request on that same table simply waits until autovacuum finishes, then runs (and typically returns almost immediately, since there's nothing left to clean). The reverse is also true — there's no scenario where both run concurrently on the same table.

---

**Q5. After `VACUUM`, can the freed space be used by future inserts on other tables, or only the same table?**

Only the same table. Space reclaimed by a plain (non-`FULL`) vacuum is marked reusable specifically for future inserts into that table — it is not returned to the OS or made available to any other table.

---

**Q6. Is `VACUUM (INDEX_CLEANUP OFF)` recommended as routine practice?**

No — it's specifically for emergency transaction ID wraparound situations, where the priority is freezing transaction IDs as fast as possible, not maintaining index performance. It should always be followed by a manual `REINDEX` once the emergency has passed; it is not a general performance optimization.
