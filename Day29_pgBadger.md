# pgBadger, EXPLAIN Plans & Query Tuning — Session Notes

This session installs and configures pgBadger (PostgreSQL's log-analysis reporting tool), walks through `EXPLAIN` vs `EXPLAIN ANALYZE`, generates and reads a real pgBadger report to find slow queries, then tunes two of those queries live — one by raising `work_mem` to eliminate a disk-based sort, one by adding an index to eliminate a sequential scan — and closes with a look at forcing alternate join methods and a practical pattern for per-user statement auditing without touching the shared application account.

---
## Table of Contents

- [1. Installing pgBadger](#1-installing-pgbadger)
- [2. Logging Prerequisites for a Useful Report](#2-logging-prerequisites-for-a-useful-report)
- [3. EXPLAIN vs EXPLAIN ANALYZE](#3-explain-vs-explain-analyze)
- [4. Generating a pgBadger Report](#4-generating-a-pgbadger-report)
- [5. Reading the Report](#5-reading-the-report)
- [6. pgBadger Is Not an AWR/ADDM Equivalent](#6-pgbadger-is-not-an-awraddm-equivalent)
- [7. Fixing a Slow Query — work_mem and External Merge Sort](#7-fixing-a-slow-query--work_mem-and-external-merge-sort)
- [8. Fixing a Slow Query — Adding an Index](#8-fixing-a-slow-query--adding-an-index)
- [9. Forcing Alternate Join Methods](#9-forcing-alternate-join-methods)
- [10. Plan Nodes Overview](#10-plan-nodes-overview)
- [11. Auditing Without Touching the Shared Application User](#11-auditing-without-touching-the-shared-application-user)
- [12. Questions Discussed in This Session](#12-questions-discussed-in-this-session)

---

## 1. Installing pgBadger

```bash
wget <pgbadger-source-url>
unzip pgbadger-master.zip
cd pgbadger-master
perl Makefile.PL      # requires the Perl module — install if missing
dnf install perl-<missing-module>   # if the Makefile.PL step errors out
perl Makefile.PL
make
make install
```

```bash
pgbadger --help    # confirms installation succeeded
```

---

## 2. Logging Prerequisites for a Useful Report

pgBadger works entirely from the PostgreSQL alert log — its output is only as good as what's actually being logged. Recommended settings before generating a report:

```ini
# postgresql.conf
log_line_prefix = '...'          # a detailed prefix pgBadger's parser expects
log_min_duration_statement = 1000  # log any statement taking over 1 second
log_connections = on
log_disconnections = on
log_checkpoints = on
```

```bash
pg_ctl restart -D /u01/pgsql18
```

---

## 3. EXPLAIN vs EXPLAIN ANALYZE

```sql
EXPLAIN SELECT * FROM employee;
```
Shows the **plan only** — what the optimizer intends to do (e.g. `Seq Scan on employee`) — without actually running the query.

```sql
EXPLAIN ANALYZE SELECT * FROM employee;
```
**Actually executes the query** and reports the real plan **plus** real timing and buffer statistics:
- **Buffers: shared hit** (served from memory) vs **shared read** (had to go to disk).
- **Planning Time** (time to produce the plan) vs **Execution Time** (time to actually run the query).

**Important limitation confirmed directly:** `EXPLAIN ANALYZE` is meaningful for `SELECT`s in the sense of "run it once and show me the real numbers," but running it on a data-modifying statement (`DELETE`, `UPDATE`) means that statement **actually executes** — demonstrated live by running `EXPLAIN ANALYZE DELETE FROM employee;`, which genuinely deleted every row (confirmed by a follow-up `SELECT` returning nothing). **`EXPLAIN` alone does not execute anything; `EXPLAIN ANALYZE` always executes the statement for real**, regardless of statement type.

---

## 4. Generating a pgBadger Report

Simulate realistic load first, then generate the report from the alert log:

```bash
pgbench -i -s 5 postgres      # seed a pgbench-style dataset
pgbench -c 10 -T 20 postgres  # simulate traffic to populate the log
```

```bash
pgbadger -o /home/postgres/log/report.html /u01/pgsql18/log/postgresql-*.log
```

```bash
# copy the report to a local machine for viewing, e.g. via WinSCP
```

---

## 5. Reading the Report

The report opens on an **overview page**, then has dedicated pages per activity category:

| Section | What it shows |
|---|---|
| Connections | Total connections made, broken down over the log period |
| Sessions | Sessions established, duration |
| Checkpoints | Checkpoint frequency/activity |
| Temporary files | How much temp table space was used, and by which query — confirmed live: 674MB used by the `ORDER BY` query from the tuning exercise |
| Vacuum | Any vacuum activity captured in the log window |
| Locks | Lock waits/conflicts, if captured |
| Queries / Long-running queries | Slowest statements captured, with exact durations — used directly to identify the three worst offenders in this session's demo data |

**Confirmed live:** the same three queries flagged as slow in the report were the exact queries run manually during the session (an `ORDER BY` query, a join query, and a lookup by a non-indexed column) — pgBadger correctly surfaced them from the log without any manual searching.

---

## 6. pgBadger Is Not an AWR/ADDM Equivalent

**Explicitly clarified, for anyone with an Oracle background:** pgBadger is **not** PostgreSQL's version of AWR, ADDM, or SQL Tuning Advisor. It gives you raw facts (which queries were slow, how long checkpoints took, how much temp space was used) — **it gives no recommendations whatsoever.** Interpreting the data and deciding what to fix is entirely manual.

**Also confirmed: there's no native snapshot-comparison mechanism** in PostgreSQL the way Oracle's AWR snapshots let you diff two points in time. Comparing "yesterday's report" against "today's report" for plan drift or performance regression has to be done manually, report to report. (A third-party extension addressing this gap was mentioned as something recently seen — not evaluated or covered in this session. Cloud-managed PostgreSQL services typically provide their own built-in snapshot mechanisms, which sidesteps this on-premise limitation.)

---

## 7. Fixing a Slow Query — work_mem and External Merge Sort

**Identified problem query** (from the report, ~12 seconds):
```sql
EXPLAIN ANALYZE SELECT * FROM test_table ORDER BY a_balance;
```

**Plan showed:** `External merge Disk: ~350MB` per CPU used for the sort — meaning the `ORDER BY` couldn't fit in memory and spilled to the temporary tablespace on disk.

**Why:** sorting happens in the `work_mem` region of local memory; this instance's `work_mem` was left at its small default (4MB), forcing the sort to disk.

```sql
SET work_mem = '1GB';
EXPLAIN ANALYZE SELECT * FROM test_table ORDER BY a_balance;
```

**Result:** the sort completed entirely **in memory** — the `External merge Disk` step disappeared from the plan, and execution time dropped substantially.

**Conceptual summary given:** without enough `work_mem`, PostgreSQL has to move data out to temporary tablespace to sort it, then bring the sorted result back — extra, avoidable disk I/O. Raising `work_mem` (sized appropriately, not carelessly, since it's allocated per sort/hash operation per session) removes that round-trip.

**Noted directly:** on a properly configured production instance (parameters already sized via `PGTune` during post-installation setup, covered in an earlier session), this kind of problem is far less likely to show up in the first place — this demo intentionally used an under-tuned instance to make the effect visible.

---

## 8. Fixing a Slow Query — Adding an Index

**Identified problem query:**
```sql
EXPLAIN ANALYZE
SELECT a.id, a.a_balance, b.b_id
FROM test_a a
JOIN pgbench_accounts b ON a.aid = b.aid
WHERE a.aid = <some_value>;
```

**Plan showed:** `Seq Scan` on `test_a`, because there was no index on the `aid` column used in both the `WHERE` clause and the join condition.

```sql
CREATE INDEX ON test_a (aid);
ANALYZE test_a;
```

**Result:** the plan switched from `Seq Scan` to `Index Scan` — execution time dropped from ~2.4 seconds to ~0.1ms (well under a second) for the same query.

**Reinforced from earlier sessions:** `ANALYZE` after creating a new index (or after any significant data change) ensures the planner has current statistics to actually choose to use the new index — skipping it can mean the new index gets ignored.

---

## 9. Forcing Alternate Join Methods

PostgreSQL has three join strategies: **nested loop**, **hash join**, and **sort-merge join**. The optimizer picks one automatically — but it can be influenced (as a diagnostic/tuning experiment, not a routine practice) via session-level planner switches:

```sql
SET enable_nestloop = off;
SET enable_hashjoin = off;
SET enable_mergejoin = off;
-- toggle these on/off in combination to force the optimizer toward
-- (or away from) a specific join strategy for testing purposes
```

**Demonstrated live:** with all three joined-table indexes in place, the default (nested loop) plan was already close to optimal; forcing a hash join in this specific case was **worse**, not better — confirmed by timing each combination directly (nested loop ~5.6s equivalent scale vs. hash join alternative ~6.8s in the demo's relative comparison).

**Practical framing given:** in the ~99% of cases, the optimizer picks the genuinely best plan on its own. These `enable_*` switches are a diagnostic tool — "give it a try, sometimes it helps, sometimes it doesn't" — not something to leave permanently altered in production without a clear, measured justification.

---

## 10. Plan Nodes Overview

PostgreSQL's `EXPLAIN` output is built from roughly **40+ distinct plan "nodes"** — each representing a specific operation the executor can perform. A handful encountered directly in this session:

| Node | Meaning |
|---|---|
| `Seq Scan` | Full table scan |
| `Index Scan` | Lookup via an index |
| `Sort` | Corresponds to `ORDER BY` — tuning lever: `work_mem` |
| `Limit` | Corresponds to `LIMIT` |
| `Hash Aggregate` | Aggregation (`COUNT`, `SUM`, `AVG`, etc.) |
| `Nested Loop` / `Hash Join` / `Sort Merge Join` | The three join strategies |
| `Function Scan` | A function used as a data source in the query |
| `Gather` | Parallel query coordination |

**Reference resource named:** a detailed public reference on PostgreSQL plan nodes from **Dalibo** — described as thorough enough to print out (~70–80 pages) and comprehensive enough that fully internalizing every node would take a couple of weeks of dedicated study, not something to expect to absorb from one session.

---

## 11. Auditing Without Touching the Shared Application User

Revisits an earlier session's guidance (enable `log_statement` per individual user, not for the shared application account, to avoid flooding the log) — with a concrete pattern for teams that **share one application-level login** for both the app itself and manual DBA/developer activity, where per-individual accounts genuinely aren't practical.

```sql
-- 1. The real application user, given to the middleware/app team
CREATE USER app_user WITH PASSWORD '...';
CREATE DATABASE app_db OWNER app_user;

-- 2. A second, individually-attributable user with the SAME privileges,
--    used only for manual/DBA activity against the same database
CREATE USER a_app_user WITH PASSWORD '...';
GRANT app_user TO a_app_user;   -- inherits all of app_user's privileges

-- 3. Enable auditing ONLY on the manual-activity user, not the app account
ALTER USER a_app_user SET log_statement = 'all';
```

**Why this works:** `a_app_user` has identical effective access to the real application (via role inheritance), but is a distinct, individually-attributable login used specifically for manual work — so `log_statement = all` on it captures exactly the manual activity worth auditing, without flooding the log with the application's own high-volume traffic on `app_user`.

**Cascading grants — the `WITH ADMIN OPTION` clause:**
```sql
GRANT app_user TO a_app_user WITH ADMIN OPTION;
```
Mirrors the same concept as Oracle's admin-option role grants: without `WITH ADMIN OPTION`, `a_app_user` can use the inherited privileges but cannot re-grant `app_user`'s role to a third user — with it, they can, letting privilege delegation cascade (A → B → C) without needing to go back to the original grantor (A) each time.

---

## 12. Questions Discussed in This Session

**Q1. Is pgBadger dependent on the alert log having specific parameters already enabled, or does it work with any default logging setup?**

It depends entirely on what's in the log — pgBadger can only report on what was actually logged. Without `log_min_duration_statement`, `log_connections`/`log_disconnections`, `log_checkpoints`, and a detailed `log_line_prefix` already enabled *before* the activity you want to analyze occurred, the corresponding sections of the report will simply be empty or incomplete. These settings need to be planned and enabled proactively, typically right after provisioning a new instance — not retroactively after you already want a report.

---

**Q2. Does enabling all these extra logging parameters for pgBadger put meaningful additional load/stress on the system?**

Explicitly addressed: no, not meaningfully — with one specific exception. `log_statement` (logging every single statement) is "the real culprit" for log volume and write overhead; none of the other parameters recommended for pgBadger (`log_connections`, `log_checkpoints`, `log_min_duration_statement`, etc.) generate anywhere near that volume. `log_statement` is deliberately handled separately (Section 11) precisely because of this cost/benefit tradeoff.

---

**Q3. Does `EXPLAIN ANALYZE` actually run a `DELETE` or `UPDATE` statement, or just simulate it?**

It genuinely executes it — confirmed live by running `EXPLAIN ANALYZE DELETE FROM employee;`, which actually deleted every row (verified by a subsequent `SELECT` returning no data). `EXPLAIN` alone (without `ANALYZE`) never executes anything; `EXPLAIN ANALYZE` always does, for any statement type, not just `SELECT`.

---

**Q4. If forcing a specific join method (via `enable_nestloop`/`enable_hashjoin`/etc.) sometimes makes a query worse, why would anyone use these settings?**

They're a diagnostic tool for cases where you suspect the optimizer picked a suboptimal plan for a specific query and want to test whether an alternative would genuinely perform better — not a setting to leave permanently changed. In the vast majority of cases (~99%, per the session), the optimizer's own default choice is already the best available plan; these switches are for the rare cases worth investigating, tested and reverted per query, not applied globally.
