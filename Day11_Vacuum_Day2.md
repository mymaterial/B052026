# Autovacuum Continued — Stats Gathering, Insert Vacuum & the Wraparound Problem — Session Notes

Continuing the autovacuum parameter deep-dive: how `autovacuum_naptime` is actually divided across databases, the separate threshold formula that governs automatic statistics gathering, why insert-only tables still need vacuum, and a full, carefully-built explanation of the transaction ID wraparound problem — PostgreSQL's single most serious "must never happen" failure mode — including the live monitoring query every production system should run.

---
## Table of Contents

- [1. autovacuum_naptime Is Divided Across Databases](#1-autovacuum_naptime-is-divided-across-databases)
- [2. Automatic Statistics Gathering — a Second, Separate Formula](#2-automatic-statistics-gathering--a-second-separate-formula)
- [3. Manual ANALYZE and Manual VACUUM Ignore the Formulas](#3-manual-analyze-and-manual-vacuum-ignore-the-formulas)
- [4. autovacuum_vacuum_insert_threshold / scale_factor](#4-autovacuum_vacuum_insert_threshold--scale_factor)
- [5. The Wraparound Problem](#5-the-wraparound-problem)
- [6. What VACUUM FREEZE Actually Does](#6-what-vacuum-freeze-actually-does)
- [7. The Transaction ID Pool Is Instance-Wide](#7-the-transaction-id-pool-is-instance-wide)
- [8. If Wraparound Actually Happens](#8-if-wraparound-actually-happens)
- [9. The Monitoring Query](#9-the-monitoring-query)
- [10. Reading the Percentage — the 10/30/70 Risk Bands](#10-reading-the-percentage--the-1030-70-risk-bands)
- [11. Live Demo — Watching the Percentage Rise and Fall](#11-live-demo--watching-the-percentage-rise-and-fall)
- [12. Questions Discussed in This Session](#12-questions-discussed-in-this-session)

---

## 1. autovacuum_naptime Is Divided Across Databases

**A nuance not covered in the previous session, addressed directly:** `autovacuum_naptime` (default 1 minute) isn't simply "sleep for 1 minute after every full scan" in isolation — **the naptime is effectively divided among however many real databases exist in the instance** (excluding `template0`/`template1`). **Practical implication stated directly:** the more databases an instance has, the more frequently the launcher effectively has to be checking *something*, since it's cycling through each database's own scan/sleep rhythm within that shared naptime budget.

**Reconfirmed, directly answering a repeated question:** the launcher performs a **full scan** of `pg_stat_all_tables`, then sleeps for the naptime interval, then repeats — this cycle is continuous, 24×7, regardless of how much or how little work was found on a given pass.

---

## 2. Automatic Statistics Gathering — a Second, Separate Formula

**A second, independent responsibility of autovacuum, distinct from dead-tuple cleanup:** gathering **statistics** — updating the planner's information about a table's row count, column value distributions, etc., used to choose the best query plan.

```
analyze_threshold = autovacuum_analyze_threshold + (autovacuum_analyze_scale_factor × number_of_live_rows)
```
**Defaults:** `autovacuum_analyze_threshold = 50`, `autovacuum_analyze_scale_factor = 0.1` (note: **half** the vacuum scale factor's default of 0.2).

**Worked example, 1000 live rows:**
```
analyze_threshold = 50 + (0.1 × 1000) = 150
```
Once **150 modifications** (inserts/updates/deletes combined) have accumulated on this table, it becomes a candidate for automatic statistics gathering — a **separate** trigger condition from the dead-tuple vacuum threshold, evaluated independently.

```sql
ANALYZE employee;
```
Manually triggers this — the equivalent operation to Oracle's `DBMS_STATS.GATHER_TABLE_STATS` (referenced directly, with the instructor noting the exact Oracle syntax from memory wasn't perfectly recalled, but the conceptual equivalence is exact).

**What actually happens during `ANALYZE`:** PostgreSQL samples the table and updates the `pg_stats` catalog view with current row-count estimates, column value distributions, and similar planner-relevant metadata — this is precisely what the query optimizer consults when deciding between a sequential scan, index scan, or join strategy for a given query (tying directly back to the EXPLAIN/query-tuning session).

---

## 3. Manual ANALYZE and Manual VACUUM Ignore the Formulas

**Confirmed directly, in response to a student's question:** running `ANALYZE` or `VACUUM` manually does **not** check or honor the threshold formulas at all — those formulas only govern when the **automatic** (autovacuum-triggered) version of each activity fires. A manual command forces the activity unconditionally; if there's genuinely nothing to do (e.g. no dead tuples), it simply completes immediately with no effect, rather than being blocked by an unmet threshold.

---

## 4. autovacuum_vacuum_insert_threshold / scale_factor

A third, related threshold pair, specifically for **insert-only** activity:
```
insert_threshold = autovacuum_vacuum_insert_threshold + (autovacuum_vacuum_insert_scale_factor × number_of_live_rows)
```
Same structural formula as the base vacuum threshold, but triggered by **insert volume** specifically, rather than dead-tuple count.

**The genuinely important question worked through directly: why would a table that only ever receives `INSERT`s (never `UPDATE`/`DELETE`) need vacuuming at all, if inserts never create dead tuples?**

**Answer: to avoid the wraparound problem (Section 5), not to clean up dead rows.** Even an insert-only table's rows carry `xmin` transaction IDs that need periodic "freezing" as part of the instance-wide transaction ID management — this insert-triggered vacuum pass exists specifically to keep even append-only tables' transaction ID bookkeeping current, independent of whether there's any dead-tuple cleanup work to do.

---

## 5. The Wraparound Problem

```sql
SELECT txid_current();
```
Every transaction (insert, update, delete, or any `BEGIN`/`COMMIT` block) consumes the next sequential transaction ID.

**The core, critical fact, stated directly and repeated for emphasis:** **PostgreSQL's transaction ID space is finite — the maximum is 2 billion.** Once 2 billion is reached, **the database will not accept any further inserts, updates, or deletes** — it effectively becomes **read-only**, because there is no valid next transaction ID to assign. **Reaching this state is called the wraparound problem**, and it is treated as one of the most serious operational failures a PostgreSQL instance can experience.

---

## 6. What VACUUM FREEZE Actually Does

```sql
VACUUM FREEZE employee;
```
**Precisely defined:** `VACUUM FREEZE` does **not** clean up dead tuples — that's a separate, unrelated activity. What it does is **replace the `xmin` value of live rows with a special "frozen" marker** (conceptually treated as something like a permanent, reusable placeholder, referred to loosely as "minus 1" for teaching purposes) — this effectively **releases** that row's original transaction ID number back into the available pool, without changing the row's actual data or its status as a live row.

**The pool-reuse mechanic, worked through with a simplified numeric example** (imagine only 10 total transaction IDs available instead of 2 billion): once several old transaction IDs are frozen, those specific numbers become available again for brand-new transactions — the *count* of available IDs is what's being protected and cycled, not any specific number staying permanently retired.

**Confirmed directly:** even though frozen rows' `xmin` no longer holds their original transaction ID, `SELECT txid_current()` will **continue to show an ever-increasing number** at the session/instance level — the "reuse" is invisible from that vantage point; it only manifests inside individual tables' actual stored `xmin` values, which you cannot directly observe without specialized tooling (the `pageinspect` extension mentioned in the previous session).

**Explicitly confirmed: freeze activity itself is a genuinely time-consuming operation** — it can take anywhere from minutes to several hours, depending on data volume, and consumption of transaction IDs continues to happen *while* freezing is in progress, which is exactly why simply "reaching the threshold" doesn't cause an instantaneous drop back to a low percentage (Section 11).

---

## 7. The Transaction ID Pool Is Instance-Wide

**Confirmed directly, in response to a direct question:** the 2-billion transaction ID ceiling operates at the **instance** level, not per-database. If an instance hosts 5 databases, they all draw from — and all need to be protected against exhausting — the **same shared** transaction ID pool. A wraparound risk in any one database, left unmanaged, threatens the entire instance.

---

## 8. If Wraparound Actually Happens

**Direct, sobering framing of the consequence:** if an instance is genuinely allowed to reach 2 billion transaction IDs (which should never happen with autovacuum properly enabled and monitored), the only remedy is to manually intervene and force the freeze/unblock process — and this recovery activity **can take anywhere from a few minutes to several hours, sometimes cited as up to 10 hours** for a genuinely large, badly-managed instance. **The entire duration of that recovery is production downtime** — no new transactions of any kind can be processed until the wraparound condition is resolved.

**The practical takeaway, stated directly and repeated:** *"the best way is to keep an eye on your transaction ID utilization — every day."* This isn't a rare edge case to think about occasionally; it's a metric that should be part of routine, ongoing production monitoring, precisely because the failure mode it prevents is catastrophic and slow to recover from.

---

## 9. The Monitoring Query

```sql
SELECT datname,
       age(datfrozenxid) AS current_txid_age,
       round(100 * age(datfrozenxid) / 2000000000.0, 2) AS percent_towards_wraparound,
       round(100 * age(datfrozenxid) / 200000000.0, 2) AS percent_towards_emergency_vacuum
FROM pg_database;
```

**The two output columns explained directly:**
- **Percentage toward wraparound** — how close this database is to the hard 2-billion ceiling. This is the number that must **never approach 100**.
- **Percentage toward emergency (autovacuum-forced) vacuum** — measured against a smaller reference point, **200 million** (10% of the full 2 billion) — because autovacuum's own automatic, aggressive freeze behavior is designed to kick in well before the real ceiling is reached, specifically to prevent the real number from ever getting dangerously close.

**Confirmed directly:** the reported "current transaction ID"-style number in this query is **not** the same as the live, ever-increasing value from `SELECT txid_current()` — it's an internally-tracked "age since last frozen" figure specific to wraparound monitoring, computed relative to `datfrozenxid`, not a raw absolute transaction counter.

---

## 10. Reading the Percentage — the 10/30/70 Risk Bands

**Direct, practical guidance on how to interpret the "percentage toward emergency vacuum" figure day-to-day:**

- **Reaching 100 of this scaled figure (i.e. ~10% of the true 2-billion ceiling) is the trigger point** where autovacuum's own aggressive/forced freeze behavior activates automatically — this is expected, normal behavior, not a crisis on its own.
- **Fluctuating a bit above 100 (into the 110–150 range) immediately after that trigger is also normal** — freeze activity itself takes time, and ongoing transaction consumption continues during that window, so a brief overshoot before the number comes back down is expected, not alarming.
- **The genuinely important operational bands, stated directly:** you don't need to actively worry until this figure approaches **30%** of the way to true wraparound; **30–70% is the "crucial" zone** requiring closer attention; **once 70% is breached, aggressive, all-hands intervention is required** — at that point, resolving it takes priority over essentially all other work until the number is brought back down below 10%.

---

## 11. Live Demo — Watching the Percentage Rise and Fall

**Demonstrated live using a purpose-built extension that artificially simulates rapid transaction consumption** (not a normal production tool — used specifically to compress a process that would normally take a long time into something observable within the session):
- At a **moderate simulated consumption rate**, the percentage climbed to roughly the 10% trigger point, autovacuum's freeze activity engaged, and the number came back down close to baseline relatively cleanly.
- At a **much higher simulated consumption rate**, the percentage significantly overshot the 10% trigger — climbing toward 40%+ — before freeze activity caught up and brought it back down, directly illustrating why sustained high write volume without adequate autovacuum resourcing (worker count, thresholds) is genuinely dangerous: the "safety margin" between the 10% auto-trigger point and true wraparound can erode faster than freezing can keep pace.

**Confirmed directly, addressing a question about disk space:** this entire mechanism has **nothing to do with disk consumption** — transaction ID usage/wraparound is a purely numeric, bookkeeping concern, entirely independent of how much actual data storage the demo's simulated activity consumed.

---

## 12. Questions Discussed in This Session

**Q1. Does the reported "percentage toward wraparound" figure directly correspond to the number you'd get from `SELECT txid_current()`?**

No — confirmed directly: the monitoring query's figure is computed from `age(datfrozenxid)`, an internally-tracked value specific to freeze/wraparound bookkeeping, distinct from the live, ever-increasing session-level transaction counter you'd see from `txid_current()`. They serve different purposes and shouldn't be conflated.

---

**Q2. If a table only ever receives `INSERT`s and never `UPDATE`/`DELETE`, does it still need vacuum?**

Yes — not for dead-tuple cleanup (which genuinely wouldn't apply to a pure insert workload), but specifically to keep that table's rows' transaction IDs current against the wraparound-prevention freeze cycle. This is exactly why `autovacuum_vacuum_insert_threshold`/`scale_factor` exist as a distinct trigger condition, separate from the dead-tuple-based threshold.

---

**Q3. If freeze activity starts as soon as the 10% threshold is crossed, why does the monitored percentage sometimes continue climbing past that point before coming back down?**

Because freeze activity is not instantaneous — it can take anywhere from minutes to hours depending on data volume, and transaction consumption continues to happen concurrently while freezing is in progress. Under moderate write load the overshoot is small and self-corrects quickly; under heavy sustained write load, the overshoot can be substantial, which is exactly why monitoring and prompt attention in the 30–70% range matters.

---

**Q4. Does the transaction ID wraparound limit apply per database, or across the whole instance?**

Across the whole instance — confirmed directly. All databases within one PostgreSQL instance draw from the same shared pool of up to 2 billion transaction IDs; there's no per-database isolation of this resource, so unmanaged transaction ID growth in any single database within a multi-database instance is a risk to the entire instance, not just that one database.
