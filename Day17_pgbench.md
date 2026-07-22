# pgbench — Reference Guide

`pgbench` is PostgreSQL's built-in benchmarking tool, shipped with the standard client tools (`/usr/pgsql-XX/bin/pgbench`). It generates a realistic OLTP-style workload (or a fully custom one you write yourself) against a real database and reports back throughput and latency — the standard way to load-test a PostgreSQL instance before trusting it with production traffic, or to measure the effect of a configuration change.

---
## Table of Contents

- [1. What pgbench Actually Does](#1-what-pgbench-actually-does)
- [2. Initializing the Benchmark Schema](#2-initializing-the-benchmark-schema)
- [3. The Four pgbench Tables](#3-the-four-pgbench-tables)
- [4. Running a Benchmark](#4-running-a-benchmark)
- [5. Reading the Output](#5-reading-the-output)
- [6. The Default Built-In Script](#6-the-default-built-in-script)
- [7. Writing a Custom Script](#7-writing-a-custom-script)
- [8. Running a Custom Script](#8-running-a-custom-script)
- [9. Other Flags Worth Knowing](#9-other-flags-worth-knowing)
- [10. Practical Guidance](#10-practical-guidance)

---

## 1. What pgbench Actually Does

At its core, `pgbench` does three things:
1. **Optionally initializes** a standard benchmark schema (4 tables, prefilled with data).
2. **Spins up N concurrent client connections**, each repeatedly running a transaction script against the database.
3. **Reports throughput** (transactions per second) and **latency** once the run completes.

It can run its own built-in default transaction script (a simplified TPC-B-style workload) or **any custom SQL script you supply** — meaning it's equally useful for generic load testing and for simulating your *actual* application's transaction mix.

---

## 2. Initializing the Benchmark Schema

```bash
pgbench -i postgres
```
Creates the 4 standard pgbench tables in the `postgres` database and populates them at the **default scale factor of 1**.

```bash
pgbench -i -s 5 postgres
```
Same tables, but at **scale factor 5** — every table's row count (`pgbench_accounts` in particular) is multiplied by 5 relative to the default. Scale factor is the standard way to control how much data you're testing against — a higher scale factor means a larger dataset and more realistic (or more stressful) I/O behavior, since a bigger dataset is less likely to fit entirely in `shared_buffers`/OS cache.

**`-i` must be run once before any benchmark test** — running a benchmark against a database that was never initialized will fail with missing-table errors.

---

## 3. The Four pgbench Tables

At the default scale factor (1), initialization creates:

| Table | Default row count (scale factor 1) | Purpose |
|---|---|---|
| `pgbench_accounts` | 100,000 | The "customer accounts" being debited/credited |
| `pgbench_branches` | 1 | Bank branches |
| `pgbench_tellers` | 10 | Tellers within those branches |
| `pgbench_history` | 0 (grows during the run) | Append-only log of every transaction processed |

**Every row count above scales linearly with `-s`** — scale factor 5 means 500,000 accounts, 5 branches, 50 tellers.

---

## 4. Running a Benchmark

Two mutually exclusive ways to define how long a benchmark run lasts — by a fixed transaction count, or by a fixed wall-clock duration.

```bash
pgbench -c 10 -t 100 postgres
```
**`-c 10`** — 10 concurrent simulated clients. **`-t 100`** — each client runs 100 transactions, then stops. Total transactions processed: 10 × 100 = 1,000.

```bash
pgbench -c 40 -t 200 postgres
```
A heavier version of the same pattern: 40 concurrent clients, 200 transactions each — 8,000 transactions total. Useful for comparing throughput/latency behavior at a higher concurrency level than the first test.

```bash
pgbench -c 10 -T 120 postgres
```
**`-T 120`** — instead of a fixed transaction count, run for a fixed **120 seconds**, with 10 concurrent clients continuously issuing transactions for that whole window. The total number of transactions completed becomes an *output* of the test, not an input.

```bash
pgbench -c 10 -T 120 postgres -P 3
```
Same as above, plus **`-P 3`** — print a progress report **every 3 seconds** while the test is running (current TPS, latency), rather than waiting silently until the full 120 seconds completes. Useful for watching a long-running test in real time and noticing performance degradation partway through.

---

## 5. Reading the Output

A typical run reports:
```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 5
query mode: simple
number of clients: 10
number of threads: 1
duration: 120 s
number of transactions actually processed: 48213
latency average = 24.891 ms
initial connection time = 12.345 ms
tps = 401.612345 (without initial connection time)
```

**The number that matters most for comparing runs: `tps` (transactions per second).** Latency average tells you the typical per-transaction response time under that load — the two numbers together describe the real tradeoff (a system can often be pushed to higher TPS at the cost of higher per-transaction latency).

---

## 6. The Default Built-In Script

When you don't supply `-f`, pgbench runs its own built-in default script — which is, functionally, exactly the `bench.sql` example shown below. Understanding the custom script (Section 7) is really understanding what the built-in default is already doing on your behalf.

---

## 7. Writing a Custom Script

```sql
-- bench.sql
\set aid random(1, 100000 * :scale)
\set bid random(1, 1 * :scale)
\set tid random(1, 10 * :scale)
\set delta random(-5000, 5000)
BEGIN;
UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
END;
```

**Line by line:**

- **`\set`** is pgbench's own scripting directive — it defines a variable, evaluated fresh for **every transaction** a client runs, not just once at script load time.
- **`random(min, max)`** picks a random integer in that inclusive range, generating realistic, varied input for each simulated transaction rather than hammering the same row repeatedly.
- **`:scale`** is a **built-in pgbench variable** — automatically set to whatever scale factor the target database was initialized with (matches the `-s` value used at `pgbench -i` time). This is exactly why `aid`'s range is written as `100000 * :scale` rather than a hardcoded number: it keeps the script correct regardless of which scale factor the database actually has, without needing to hardcode or manually recalculate the row counts from Section 3.
- **`:aid`, `:bid`, `:tid`, `:delta`** — referencing a variable defined via `\set`, substituted directly into the SQL that follows, the same way a bind variable would be used.
- **`BEGIN` / `END`** — wraps the whole sequence in a single transaction, exactly the kind of multi-statement, all-or-nothing unit covered in the ACID/atomicity material — this script is deliberately modeling a real banking-style transfer transaction (debit one account, credit a teller and branch balance, log the transfer), not just a single isolated statement.

**What this script is actually simulating:** a simplified version of the classic TPC-B benchmark — a customer's account balance changes, the teller and branch running totals are updated to match, and a permanent audit record is appended to `pgbench_history` — all within one atomic transaction, exactly the shape of a real-world funds-transfer operation.

---

## 8. Running a Custom Script

```bash
pgbench -c 10 -T 120 postgres -P 3 -f bench.sql
```
Identical to the earlier `-T`/`-P` example, except **`-f bench.sql`** tells pgbench to run this custom transaction script instead of its own built-in default — every one of the 10 concurrent clients repeatedly executes the full `bench.sql` transaction, with fresh random values each time, for the full 120-second window, reporting progress every 3 seconds.

**Why write a custom script at all, given the built-in default already does something similar:** the real value is being able to substitute **your own application's actual queries** in place of the generic `pgbench_accounts`-style statements — pointing pgbench at your real schema and real transaction shapes turns it from a generic hardware benchmark into a genuine load test of your specific application's behavior under concurrency.

---

## 9. Other Flags Worth Knowing

| Flag | Purpose |
|---|---|
| `-j N` | Number of worker threads pgbench itself uses to drive the `-c` clients (useful to spread client load across multiple CPUs on the *machine running pgbench*, not the database server) |
| `-n` / `--no-vacuum` | Skip the automatic `VACUUM` pgbench normally runs on its tables before starting a test — useful when deliberately testing against a table in a specific bloat state |
| `-M <mode>` | Query protocol mode: `simple`, `extended`, or `prepared` — `prepared` reuses parsed/planned statements across executions and typically shows the best raw throughput numbers |
| `-r` | Report per-statement latency breakdown within the script, not just the overall transaction latency |
| `-U <user>` / `-h <host>` / `-p <port>` | Standard connection parameters, same as `psql` |

---

## 10. Practical Guidance

- **Always initialize at a scale factor that reflects your real data volume**, not the tiny default — scale factor 1 (100,000 accounts, well under 1GB) fits comfortably in memory on almost any machine and won't meaningfully exercise disk I/O; a scale factor large enough that the dataset exceeds `shared_buffers` gives a far more realistic picture.
- **Prefer `-T` (duration-based) over `-t` (count-based) for most real testing** — a fixed duration gives directly comparable TPS figures across runs and configuration changes; a fixed transaction count means a slower configuration simply takes longer to finish, which is harder to reason about at a glance.
- **Always use `-P` on any run longer than a minute or two** — watching TPS/latency evolve over the run (e.g. degrading as `shared_buffers` fills, or as autovacuum kicks in) is often more informative than the single final summary number.
- **Use a custom `-f` script whenever the goal is evaluating a real configuration change** (e.g. before/after raising `work_mem`, adding an index, tuning `checkpoint_completion_target`) — the built-in default script is fine for a generic sanity check, but a custom script matching your actual query shapes gives a result that's actually meaningful for your workload.
