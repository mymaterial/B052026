# Background Processes Recap & Post-Installation Steps — Session Notes

Wrapping up the background-processes unit — the WAL writer/background writer/checkpointer relationship, the LRU-based flushing algorithm, `temp_buffers` and temporary tablespaces, and a rapid-fire vacuum-speed-tuning quiz — before moving into **Post-Installation Steps**: the three things every fresh PostgreSQL cluster needs before being handed to an application, demonstrated through live, deliberately-broken connection troubleshooting, PgTune-based parameter sizing, and a first Python (`psycopg2`) connectivity script.

---
## Table of Contents

- [1. pg_cron Scheduling Best Practices](#1-pg_cron-scheduling-best-practices)
- [2. Recap — the UPDATE Walkthrough, in Full](#2-recap--the-update-walkthrough-in-full)
- [3. WAL Writer, Background Writer, Checkpointer — Who Does What](#3-wal-writer-background-writer-checkpointer--who-does-what)
- [4. The LRU Flushing Algorithm](#4-the-lru-flushing-algorithm)
- [5. Checkpointer Is a Watchman *and* a Worker](#5-checkpointer-is-a-watchman-and-a-worker)
- [6. Monitoring buffers_checkpoint vs. buffers_clean](#6-monitoring-buffers_checkpoint-vs-buffers_clean)
- [7. temp_buffers and Temporary Tablespaces](#7-temp_buffers-and-temporary-tablespaces)
- [8. Quiz — Making Autovacuum Finish Faster](#8-quiz--making-autovacuum-finish-faster)
- [9. Quiz — Rebalancing Checkpointer vs. Background Writer](#9-quiz--rebalancing-checkpointer-vs-background-writer)
- [10. Post-Installation Step 1 — Set the postgres Password](#10-post-installation-step-1--set-the-postgres-password)
- [11. Post-Installation Step 2 — Enable External Connectivity](#11-post-installation-step-2--enable-external-connectivity)
- [12. The Postmaster/Listener/Rulebook Mental Model](#12-the-postmasterlistenerrulebook-mental-model)
- [13. pg_hba.conf Is Read Top to Bottom — Order Matters](#13-pg_hbaconf-is-read-top-to-bottom--order-matters)
- [14. The Four Connection Error Messages, Decoded](#14-the-four-connection-error-messages-decoded)
- [15. Post-Installation Step 3 — PgTune Parameter Sizing](#15-post-installation-step-3--pgtune-parameter-sizing)
- [16. Connection Strings](#16-connection-strings)
- [17. First Python (psycopg2) Script](#17-first-python-psycopg2-script)
- [18. Questions Discussed in This Session](#18-questions-discussed-in-this-session)

---

## 1. pg_cron Scheduling Best Practices

**Confirmed directly, clarifying a scope question:** `pg_cron` jobs run **at the database level**, not the instance level — connecting to `postgres` (with no database specified) runs a job against that specific database only. To run a job against a different database, you must name it explicitly; to run the same job across every database in the instance, you have to iterate through them yourself (e.g. looping via a script, or writing separate scheduled entries per database) — there's no built-in "run on all databases" shortcut.

**Scheduling guidance for maintenance-type jobs (vacuum, backups, report generation), given directly:**
- Run during **non-business hours**, and specifically avoid overlapping with any *other* scheduled job (backups, materialized view refreshes, report generation, etc.) — **no two maintenance-type jobs should run concurrently.**
- **Reasoning given directly:** `VACUUM` itself does not block other activity, but it is frequently *blocked by* other things (most notably long-running transactions, covered in depth in the prior autovacuum sessions) — so the goal of careful scheduling is to avoid creating exactly the kind of contention that would stall vacuum, not to protect other jobs from vacuum.

**A concrete tuning pattern for a quiet-hours window, given directly:** if monitoring shows CPU utilization is very low (e.g. 5–10%) during a known off-peak window (e.g. 3–5 AM), that window can be used to temporarily raise `maintenance_work_mem` (suggested: 10–20% of total RAM) and increase the effective worker/CPU allocation for maintenance activity (suggested: roughly a quarter of total CPU count) — worked through with an example 16-vCPU/32GB RAM machine, landing on roughly 8GB for `maintenance_work_mem` and about 6 CPUs' worth of worker affinity for that window specifically.

---

## 2. Recap — the UPDATE Walkthrough, in Full

A full restatement of the `UPDATE` architecture walkthrough from the earlier dedicated session, as a lead-in to today's deeper dive into the remaining background processes:

1. Copy the existing row within the table.
2. Bring it to memory, modify it.
3. Client sees the "1 row updated" confirmation.
4. Sometime later, the **background writer** flushes the modified page from `shared_buffers` to the data file.
5. **Checkpoint activity** ensures there are no remaining dirty buffers, and updates the control file accordingly.

---

## 3. WAL Writer, Background Writer, Checkpointer — Who Does What

Three genuinely distinct jobs, reinforced with precise, one-line definitions:

- **WAL writer:** writes information from `wal_buffers` to the physical WAL file, **on every commit.**
- **Background writer:** writes information from `shared_buffers` to physical data files, **continuously, 24×7**, at its own pace.
- **Checkpointer:** ensures there are **no dirty buffers** left in `shared_buffers`, on a periodic cycle (every 5 minutes by default) — its *primary* role is verification/coordination, not writing.

**Confirmed directly, in response to a question about rollback and WAL:** WAL only ever records **committed** transaction information — a rolled-back transaction's changes are never written to the WAL file at all, since WAL exists specifically to support crash recovery and replay of durable (committed) changes.

---

## 4. The LRU Flushing Algorithm

**The background writer's actual per-cycle behavior, walked through concretely:**

```ini
bgwriter_delay = 200ms          # sleep interval between bursts
bgwriter_lru_maxpages = 100     # max pages written per burst (default)
```

Given, say, 250 dirty buffers needing to be flushed: the background writer writes up to `bgwriter_lru_maxpages` pages (100 by default), sleeps for `bgwriter_delay` (200ms), wakes up, writes the next batch, sleeps again, and so on — repeating this burst-then-sleep cycle continuously.

**Confirmed directly, in response to a question:** this is the same **LRU (Least Recently Used)** algorithm concept used by Oracle's own buffer-flushing mechanism — PostgreSQL's background writer decides *which* dirty pages to flush first using the same underlying "least recently used" prioritization logic, not an arbitrary or random selection.

---

## 5. Checkpointer Is a Watchman *and* a Worker

**A distinction drawn out carefully in response to student confusion:** the checkpointer's stated *primary* role is to **verify** that `shared_buffers` has no dirty pages remaining and, if it finds none, simply update the control file and go back to sleep — a "watchman" role. **But the checkpointer does not delegate cleanup to the background writer if it finds dirty buffers still present** — it will **write those dirty buffers to disk itself**, directly, rather than notifying or invoking the background writer to do it.

**Explicitly reinforced, correcting a direct misconception:** the checkpointer never "invokes" the background writer — there is no inter-process signaling between them, consistent with the earlier "nobody informs nobody" correction from the INSERT/DELETE session. Each process independently checks and acts on the same shared memory state.

**Why this distinction matters operationally, stated directly:** if the checkpointer is frequently doing substantial dirty-buffer writing itself (rather than finding the room already "empty" thanks to the background writer keeping up), that's a sign of **bad configuration** — the checkpointer performing this work is genuinely CPU-intensive and expensive, and a well-tuned system should have the background writer keeping pace well enough that the checkpointer rarely finds significant cleanup work waiting for it.

---

## 6. Monitoring buffers_checkpoint vs. buffers_clean

```sql
SELECT * FROM pg_stat_bgwriter;
```

**Two key counters, explained directly:**
- **`buffers_checkpoint`** — how many dirty buffers the **checkpointer itself** has had to write to disk.
- **`buffers_clean`** — how many dirty buffers the **background writer** has written to disk.

**The health rule, stated directly and repeated: `buffers_checkpoint` should always be low — ideally near zero — relative to `buffers_clean`.** If `buffers_checkpoint` is consistently high (the checkpointer is doing significant direct write work), that indicates the background writer isn't keeping the buffer pool clean fast enough on its own, and tuning (`bgwriter_delay`, `bgwriter_lru_maxpages`) is warranted.

**Live demonstration of this exact scenario, deliberately provoked:** after generating a burst of activity and immediately issuing a manual `CHECKPOINT`, `buffers_checkpoint` incremented directly — proving the checkpointer had to do the write work itself in that moment, because the background writer (a "lazy" process, by design) hadn't yet gotten to those specific dirty pages.

**Direct confirmation, addressing a question about data integrity:** yes, a checkpoint writes **both committed and uncommitted** data present in `shared_buffers` to the data file — this is expected and not a correctness problem, because **`VACUUM` is what later cleans up any uncommitted/dead data** that ends up physically written this way; the checkpoint mechanism itself doesn't distinguish between committed and uncommitted content when flushing.

---

## 7. temp_buffers and Temporary Tablespaces

**Two remaining local memory components not yet covered in depth:** `work_mem` (used for sorting) and `temp_buffers` (used for temporary tables/temporary tablespace).

**Live demo — a sort spilling to disk:**
```sql
EXPLAIN ANALYZE SELECT * FROM pgbench_accounts ORDER BY abalance;
-- "Sort Method: external merge  Disk: ..."
```
Confirmed: with `work_mem` insufficient for the sort, PostgreSQL falls back to using the **temporary tablespace on disk** rather than sorting entirely in memory.

```sql
SET work_mem = '2GB';
EXPLAIN ANALYZE SELECT * FROM pgbench_accounts ORDER BY abalance;
-- "Sort Method: quicksort  Memory: ..."
```
With `work_mem` raised sufficiently, the same sort completes entirely **in memory** — directly confirming `work_mem`'s role as the tuning lever for exactly this behavior (consistent with the earlier pgBadger/query-tuning session's `work_mem` discussion).

**Temporary tables and `temp_buffers`, demonstrated live:**
```sql
CREATE TEMPORARY TABLE department (id int, salary int);
```
A temporary table lives **only for the duration of the session** — once disconnected, it's gone. **Where it actually lives, precisely:** if the table's data fits within `temp_buffers`, it stays there, in memory. **If it doesn't fit, PostgreSQL creates actual temporary files on disk**, inside the physical temporary tablespace directory (confirmed live by navigating to `base/pgsql_tmp/` and observing files being created and removed as a temp table's activity happened, then completed).

**Confirmed directly, addressing a sizing question:** there's no formula or percentage-of-dataset guideline for `temp_buffers` — practical guidance given was a flat cap around **32MB**, regardless of overall data volume, since anything genuinely needing more than that will spill to disk anyway and doesn't benefit meaningfully from a larger in-memory allocation.

**Confirmed directly: table spaces are logical, data files are physical** — reinforcing the general PostgreSQL storage-hierarchy vocabulary (tablespace as the logical container, data files as what's actually on disk within it) already used for regular tables, applying identically to the temporary tablespace.

---

## 8. Quiz — Making Autovacuum Finish Faster

**Posed directly as a single-parameter quiz question: "I want autovacuum on a given table to finish faster — which one parameter should I change?"**

**Answer, worked through by elimination:**
- **Not `autovacuum_max_workers`** — that parameter only controls *how many CPUs/tables* can be vacuumed in parallel, not how fast any single vacuum operation itself runs.
- **Correct answer: `autovacuum_work_mem`** (or, absent that, `maintenance_work_mem`) — since vacuum activity happens entirely in this memory region, a larger allocation directly speeds up a single vacuum pass.
- **Also correct, as an additional lever: the cost-based throttling parameters** — `autovacuum_vacuum_cost_delay` (reduce toward 0, meaning "don't sleep between bursts") and/or `vacuum_cost_limit`/`autovacuum_vacuum_cost_limit` (increase, meaning "do more work before pausing") — both directly speed up a single vacuum's wall-clock completion time, at the cost of more sustained I/O/CPU pressure while it runs.

**Explicit tradeoff stated directly:** these are genuine speed/impact tradeoffs, not free wins — a faster vacuum via reduced throttling means more concurrent resource contention with regular application traffic while it's running.

---

## 9. Quiz — Rebalancing Checkpointer vs. Background Writer

**A parallel, follow-on quiz: "I've observed the checkpointer is writing more buffers than the background writer — which parameters should I adjust, and in which direction?"**

**Worked through directly, as two coordinated changes:**
- **Increase `bgwriter_lru_maxpages`** — let the background writer flush more pages per burst, so it keeps up better and leaves fewer dirty buffers for the checkpointer to find.
- **Decrease `bgwriter_delay`** (potentially toward 0) — have the background writer sleep less between bursts, writing more continuously.

**Explicit caution given directly, twice:** don't blindly set `bgwriter_delay` to 0 in production without first monitoring — doing so means the background writer is continuously writing with essentially no pause, which is itself a real resource-consumption tradeoff, not a free performance win. The right values depend on observed system behavior (via `pg_stat_bgwriter`), not a fixed recommendation to apply universally.

**Direct confirmation of why an overloaded checkpointer is a genuine problem, not just a curiosity:** a checkpointer doing excessive direct write work is itself CPU-intensive, and can manifest as "checkpoints occurring too frequently" warnings in the log, or checkpoint activity failing to complete/update the control file promptly — both real, actionable production symptoms.

---

## 10. Post-Installation Step 1 — Set the postgres Password

**Stated directly as the very first thing to do, before handing a fresh cluster to any application:** by default, the `postgres` superuser has **no password at all**.

```sql
\password
Enter new password for user "postgres": ********
Enter it again: ********
```

---

## 11. Post-Installation Step 2 — Enable External Connectivity

**Demonstrated live, deliberately unconfigured first, to show the actual failure progression a fresh cluster produces.** Three separate changes are required, confirmed as genuinely independent layers (matching the pattern from earlier sessions, now shown as a single coherent "post-install" checklist):

**2.1 — `postgresql.conf`:**
```ini
listen_addresses = '*'
```
Requires a **restart** (confirmed live: `pg_ctl restart -D /u01/pgsql18`).

**2.2 — `pg_hba.conf`:**
```
host    all    all    0.0.0.0/0    md5
```
Requires only a **reload** (`SELECT pg_reload_conf();`), not a restart.

**2.3 — OS-level firewall:**
```bash
systemctl stop firewalld
# or, in production:
firewall-cmd --zone=public --permanent --add-port=5432/tcp
firewall-cmd --reload
```

**Explicit, repeated production guidance:** disabling the firewall outright (`systemctl stop firewalld`) is used throughout the course purely for lab convenience — **in a real production environment, you must open the specific required port** rather than disabling the firewall wholesale.

---

## 12. The Postmaster/Listener/Rulebook Mental Model

**A conceptual model built directly to explain the layered connection flow, illustrated live with two example client hosts (10.4 and 10.5):**

1. **The OS-level firewall** is the first barrier — a client's connection attempt never even reaches PostgreSQL's own logic if the port itself is blocked.
2. **`listen_addresses`** determines whether the postmaster's listener is paying attention to a given interface/IP at all — set to `*`, it listens on every interface; set to a specific IP, only that one; set to `localhost`, external connections are ignored entirely regardless of firewall/`pg_hba.conf` state.
3. **`pg_hba.conf`** is "the rulebook" — once a connection attempt reaches the listener, this file determines whether that specific client/database/user combination is actually allowed through, and by what authentication method.

**Demonstrated live with a real allow/reject pair:** with a rule permitting connections from any host (`0.0.0.0/0`) to database A, a request from host 10.4 for database A succeeds (after a password prompt); a hypothetical request that doesn't match any configured rule would be rejected — directly illustrating that `pg_hba.conf` evaluation is about matching a specific request against specific configured rules, not a blanket allow-or-deny toggle.

---

## 13. pg_hba.conf Is Read Top to Bottom — Order Matters

**Demonstrated live and stated directly as a core operational fact:** the postmaster reads `pg_hba.conf` **from top to bottom**, and uses the **first matching rule** it finds — later rules are never consulted once an earlier one matches.

**Practical consequence, worked through directly:** if you want to selectively **block** connections to one specific database (e.g. database A) while still allowing broader access to others, a more specific, restrictive rule for that database must be placed **above** any broader, more permissive rule that would otherwise match it first — placing a restrictive rule below a catch-all `all` rule has no effect, since the catch-all rule (appearing first) will always win.

---

## 14. The Four Connection Error Messages, Decoded

Walked through as a direct, memorable mapping — each message pinpoints exactly which layer failed:

| Message | Meaning |
|---|---|
| `connection timeout expired` / `No route to host` | Never even reached the server — an OS/network/firewall-level block, before PostgreSQL is involved at all |
| `FATAL: no pg_hba.conf entry for host "X", user "Y", database "Z"` | Reached the postmaster, but **no rule at all** exists in `pg_hba.conf` matching this specific request |
| `FATAL: pg_hba.conf rejects connection for host "X", ...` | Reached the postmaster, and a rule **did** match — but that rule's method is explicitly `reject` |
| (Connection simply hangs / times out at the listener level) | `listen_addresses` isn't configured to listen on the interface the client is arriving on at all |

**The core diagnostic principle reinforced directly:** each of these four distinct messages tells you precisely which of the three layers (firewall, listener/`listen_addresses`, `pg_hba.conf` rulebook) to go check first — there's no need to guess or check all three when the specific error message already narrows it down.

---

## 15. Post-Installation Step 3 — PgTune Parameter Sizing

```
https://pgtune.leopard.in.ua/
```

**Demonstrated live:** enter your PostgreSQL version, OS, workload type (OLTP in this demo), total RAM, CPU count, expected max connections, and storage type (SSD) — PgTune generates a full recommended parameter set (`max_connections`, `shared_buffers`, `effective_cache_size`, `maintenance_work_mem`, `checkpoint_completion_target`, `wal_buffers`, `random_page_cost`, `work_mem`, `max_wal_size`, parallel-worker settings, and more).

**Explicitly clarified, in response to a direct question:** PgTune is **not** an official PostgreSQL project — it's a well-regarded, community-built tool based on accumulated real-world tuning experience and the official documentation's own guidance, but it carries no official endorsement. **Still explicitly recommended as a reliable, practical starting point.**

**Explicitly flagged as tomorrow's topic, not covered today:** *why* PgTune recommends each specific value (the actual significance of `max_connections`, `effective_cache_size`, `maintenance_work_mem`, etc.), and which of these parameters require a restart vs. only a reload to take effect — today's session covered only *that* this step exists and *how* to generate the recommendations.

---

## 16. Connection Strings

**The standard set of connection parameters, laid out directly:** host, port, username, password, database name.

**Confirmed directly, addressing a direct question about defaults:** omitting the host defaults to `localhost`; omitting the database name defaults to a database matching the connecting **username** (not `postgres` by default, as might be assumed) — this is standard `psql`/libpq behavior, worth knowing since it can otherwise produce a confusing "database does not exist" error for anyone expecting an implicit `postgres` default.

**Explicitly noted: PostgreSQL has no concept equivalent to Oracle's TNS entries / SCAN listener addresses** — connection information is passed directly and explicitly (host, port, database, user, password) rather than resolved through any centralized naming/directory service PostgreSQL itself provides.

---

## 17. First Python (psycopg2) Script

```python
import psycopg2

connection = psycopg2.connect(
    host="192.168.108.131",
    port=5432,
    database="postgres",
    user="postgres",
    password="********"
)
print("Connected to database successfully")

cursor = connection.cursor()
cursor.execute("SELECT version();")
print(cursor.fetchone())

cursor.execute("SELECT * FROM employee;")
print(cursor.fetchall())

cursor.close()
connection.close()
```

Built up incrementally and run live, confirming: a successful connection message, retrieving the server version, and querying the `employee` table created earlier in the session.

**A live demo of AI-assisted coding tools was layered on top, purely as an aside:** using an integrated AI coding assistant (referred to as "Copilot"/"Cursor" in the session) to generate and adjust an `INSERT` script by describing the table's actual column structure in plain language — explicitly framed as optional, illustrative material, not a core teaching point. **Confirmed directly: auto-completion/suggestion tooling is already built into modern IDEs by default and requires no separate manual installation** — simply having `psycopg2` (or the equivalent library for any other language) installed is enough for an IDE's built-in suggestions to work.

---

## 18. Questions Discussed in This Session

**Q1. Is reloading the configuration file (`pg_reload_conf()` / `pg_ctl reload`) equivalent to restarting the database?**

No — confirmed directly and explicitly: reload and restart are genuinely different operations. Certain parameters (like `pg_hba.conf` rule changes) only require a reload; others (like `listen_addresses`) require a full restart. Which one a given parameter needs is documented per-parameter, not a blanket rule.

---

**Q2. If a `CHECKPOINT` writes both committed and uncommitted data to the data file, does that create a data-integrity risk?**

No — confirmed directly: this is expected, normal behavior, not a bug or risk. `VACUUM` is specifically responsible for later cleaning up any uncommitted/dead data that ends up physically written to the data file this way; the checkpoint mechanism's job is purely to ensure durability of whatever is currently in memory, not to distinguish transaction status while doing so.

---

**Q3. Does `pg_cron` support running a single scheduled job automatically across every database in an instance?**

No — confirmed directly: `pg_cron` jobs are scoped to whichever database the job's connection targets. Running the same logical job across multiple databases requires either explicit per-database job definitions or a wrapper script that iterates through the databases itself; there's no native "all databases" execution mode.

---

**Q4. If a `pg_hba.conf` rule intended to restrict access to a specific database doesn't seem to take effect, what's the most likely cause?**

Rule ordering — confirmed directly and demonstrated live: `pg_hba.conf` is evaluated top to bottom, and the **first matching rule wins**. A more specific/restrictive rule placed *below* a broader, already-matching rule will never actually be reached or enforced; the fix is to reorder the file so the more specific rule appears first.
