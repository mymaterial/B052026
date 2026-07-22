# PgTune Recommended Parameters — Reference Guide

A working reference for the parameter set PgTune (https://pgtune.leopard.in.ua/) generates. For each parameter: what it does, whether PgTune's number is usually the right one to keep as-is, and what actually changes if you tune it further.

**General framing for the session:** PgTune gives you a solid, sane *starting point* based on your declared RAM/CPU/connection count/workload type — it is not a finished, workload-verified answer. Some of these values are safe to leave alone almost indefinitely; others are genuinely just a first guess that should be revisited once you have real production monitoring data. The table below flags which is which.

---

## Memory Parameters

### `shared_buffers` — `4GB`
PostgreSQL's own shared memory cache for table and index pages — the direct equivalent of Oracle's database buffer cache.

**Change it?** Rarely, and only within a narrow band. PgTune's ~25% of RAM follows the long-standing community rule of thumb.
- **Pros of increasing:** more hot data stays cached, fewer disk reads.
- **Cons of increasing:** diminishing and eventually *negative* returns — PostgreSQL relies on the OS page cache as a second-level cache, and over-allocating `shared_buffers` starves that layer and can hurt checkpoint/writeback behavior.
- **Rule of thumb:** stay in the 20–40% of RAM range. Don't push toward 50%+ without specific evidence it helps.
- **Restart required.**

### `effective_cache_size` — `12GB`
Not an actual memory allocation — it's a **hint to the query planner** about how much total memory (shared_buffers + OS cache) is realistically available for caching data, used to judge whether an index scan is worth it.

**Change it?** Safe to leave as PgTune's estimate (typically 50–75% of RAM) in almost all cases.
- **Pros of increasing:** planner leans more toward index scans, which is usually correct if your data really is well-cached.
- **Cons of setting it too high:** planner may over-favor index scans even when data isn't actually cached, causing worse plans.
- **Rule of thumb:** 50–75% of total RAM. No real downside to getting this approximately right rather than exact.
- **Reload only — no restart.**

### `maintenance_work_mem` — `1GB`
Memory budget for maintenance operations: `VACUUM`, `CREATE INDEX`, `ALTER TABLE ... ADD FOREIGN KEY`, etc. Separate from `work_mem`.

**Change it?** Often worth raising temporarily for specific maintenance windows.
- **Pros of increasing:** faster index builds, faster vacuum on bloated tables.
- **Cons of increasing:** each concurrent maintenance operation can use up to this much — several running at once can add up fast.
- **Rule of thumb:** PgTune's default is fine day-to-day; bump to 2–4× higher only during an off-hours maintenance window, then set it back.
- **Reload only — no restart.**

### `work_mem` — `8256kB`
Memory allowed **per sort/hash operation** — and a single complex query can use several of these simultaneously (one per sort, hash join, etc.), multiplied across every concurrent session.

**Change it?** This is the one PgTune value most worth double-checking against real behavior, not just accepting.
- **Pros of increasing:** fewer sorts/hash joins spill to disk (visible in `EXPLAIN ANALYZE` as "external merge Disk").
- **Cons of increasing:** true worst-case memory use scales as `work_mem × concurrent_operations × concurrent_sessions` — set too high, this is a realistic path to an out-of-memory event under load, unlike most other parameters here.
- **Rule of thumb:** keep the global setting conservative (PgTune's number is usually reasonable); raise it **per-session** (`SET work_mem = '256MB';`) for specific reporting/analytical queries instead of raising it globally.
- **Reload only — no restart.**

### `wal_buffers` — `16MB`
Shared memory holding WAL data that hasn't yet been written to the WAL file.

**Change it?** Essentially never needs manual attention. PgTune's value matches PostgreSQL's own auto-tuning behavior (`-1` auto-sizes to ~3% of `shared_buffers`, capped around 16MB).
- **Rule of thumb:** leave it. Not a lever worth spending time on.
- **Restart required** (if changed explicitly).

### `huge_pages` — `try`
Whether PostgreSQL should use the OS's huge memory pages for its shared memory segment (reduces CPU overhead from memory-address translation on large `shared_buffers`).

**Change it?** `try` is the safe default — attempts huge pages, silently falls back if the OS hasn't got them configured. Only meaningfully "on" if the OS-level huge pages are actually pre-allocated (`vm.nr_hugepages`) — that's a separate, OS-level step this parameter alone doesn't do for you.
- **Rule of thumb:** leave as `try` unless you've specifically configured OS huge pages and want startup to hard-fail if they're unavailable (`on`).
- **Restart required.**

---

## Checkpoint & WAL Parameters

### `checkpoint_completion_target` — `0.9`
The fraction of the time between checkpoints over which checkpoint I/O should be spread out, rather than dumped all at once.

**Change it?** No — `0.9` (spread checkpoint writes across 90% of the checkpoint interval) is already the community best-practice value. There's little reason to move it.
- **Reload only — no restart.**

### `min_wal_size` — `2GB` / `max_wal_size` — `8GB`
`max_wal_size` is the soft ceiling that triggers a checkpoint once crossed — the main lever controlling **how often checkpoints happen**. `min_wal_size` keeps a floor of recycled WAL segments on hand so PostgreSQL doesn't keep creating/deleting WAL files under light load.

**Change it?** `max_wal_size` is genuinely worth raising further on write-heavy OLTP systems.
- **Pros of increasing `max_wal_size`:** fewer, less frequent checkpoints → smoother write throughput, less I/O spiking.
- **Cons of increasing `max_wal_size`:** longer crash recovery time (more WAL to replay since the last checkpoint) and more disk space consumed by WAL between checkpoints.
- **Rule of thumb:** PgTune's numbers are a reasonable floor; on a write-heavy system with disk space to spare, doubling `max_wal_size` (e.g. 16GB) is a common, safe next step if checkpoint-frequency warnings show up in the log.
- **Reload only — no restart, for either.**

### `wal_compression` — `lz4`
Compresses full-page images written to WAL, trading a small amount of CPU for less WAL volume and less WAL-related disk I/O.

**Change it?** Leave it on. `lz4` specifically is the fast, low-CPU-overhead choice (vs. the older, slower `pglz`).
- **Rule of thumb:** keep enabled on any I/O-constrained production system; only disable if CPU is your actual bottleneck, not I/O.
- **Reload only — no restart.**

### `io_method` — `io_uring`
(PostgreSQL 18+) Selects the underlying mechanism used for asynchronous I/O — options are `sync`, `worker`, and `io_uring`.

**Change it?** Only if your platform doesn't support it.
- **Requirement:** Linux-only, needs a reasonably modern kernel. On non-Linux systems or older kernels, this setting isn't valid and `worker` should be used instead.
- **Rule of thumb:** keep `io_uring` on modern Linux; fall back to `worker` elsewhere.
- **Restart required.**

---

## Planner Cost Parameters

### `random_page_cost` — `1.1`
The planner's assumed cost of a *random* disk page fetch, relative to a sequential one (`seq_page_cost = 1.0` by default).

**Change it?** This is a storage-hardware-dependent value, and PgTune's `1.1` assumes SSD/NVMe.
- **Pros of the low value:** on real SSD/NVMe storage, random and sequential access genuinely cost about the same — this makes the planner correctly favor index scans more often.
- **Cons:** if you're actually running on spinning HDD storage, `1.1` is wrong and will cause the planner to over-favor index scans that are genuinely more expensive there — the historical HDD-era default of `4.0` is more appropriate in that case.
- **Rule of thumb:** `1.1`–`1.2` for SSD/NVMe (most modern deployments); leave near `4.0` for HDD.
- **Reload only — no restart.**

### `effective_io_concurrency` — `200`
How many concurrent I/O requests the planner should assume the storage can usefully handle at once — affects prefetching behavior (e.g. bitmap heap scans).

**Change it?** Match it to real storage capability.
- **Rule of thumb:** `200`+ is reasonable for NVMe/high-end SSD with real parallel I/O capacity; drop to something like `2` for spinning disk. Has no effect on Windows.
- **Reload only — no restart.**

### `default_statistics_target` — `100`
Controls how much sampling detail `ANALYZE` collects per column, which the planner then uses for row-count/selectivity estimates.

**Change it?** `100` is already PostgreSQL's own default — PgTune isn't changing anything here.
- **Pros of raising it (e.g. to 500):** more accurate planner estimates for large tables or columns with complex/skewed value distributions.
- **Cons of raising it:** slower `ANALYZE` runs, larger `pg_statistic` catalog entries.
- **Rule of thumb:** leave the global default at `100`; raise it **per-column** (`ALTER TABLE ... ALTER COLUMN ... SET STATISTICS 500;`) only for specific columns where you've actually observed bad plans from poor estimates.
- **Reload only — no restart.**

---

## Parallelism & JIT

### `max_worker_processes` — `8`
The total pool of background worker processes available for the whole instance — parallel query workers, logical replication workers, and extension-owned workers (e.g. `pg_cron`) all draw from this single pool.

**Change it?** Scale it to real CPU count and how many extensions/replication workers you're actually running.
- **Rule of thumb:** should be ≥ your CPU count, and must be ≥ `max_parallel_workers`. If you add logical replication or several worker-based extensions later, revisit this number.
- **Restart required.**

### `max_parallel_workers` — `8` / `max_parallel_workers_per_gather` — `4`
`max_parallel_workers` is the instance-wide cap on workers actively running parallel queries at once; `max_parallel_workers_per_gather` caps how many workers a **single** query can request.

**Change it?** Usually fine as PgTune sets them (`max_parallel_workers` ≈ CPU count).
- **Cons of raising `max_parallel_workers_per_gather` too high:** a handful of concurrent large queries can each grab many workers and starve everything else running on the system.
- **Rule of thumb:** leave `max_parallel_workers` near CPU count; only raise `max_parallel_workers_per_gather` above 4 on a system genuinely dominated by a few large analytical queries rather than many concurrent small ones.
- **Reload only — no restart, for either** (bounded by `max_worker_processes`, which does need a restart).

### `max_parallel_maintenance_workers` — `4`
Caps parallelism specifically for maintenance operations that support it (mainly `CREATE INDEX` and some `VACUUM` scenarios).

**Change it?** Fine to raise temporarily for a big index-build maintenance window, same logic as `maintenance_work_mem`.
- **Reload only — no restart.**

### `jit` — `off`
Just-In-Time compilation of query expressions — can speed up long, CPU-heavy analytical queries, but adds compilation overhead to every query it applies to.

**Change it?** PgTune disabling this is a deliberate, common choice for OLTP-shaped workloads (lots of short queries), not an oversight.
- **Pros of turning on:** meaningful speedup for long-running analytical/OLAP-style queries.
- **Cons of turning on:** the compilation overhead itself can make short, frequent OLTP queries *slower*, not faster.
- **Rule of thumb:** keep `off` for typical OLTP systems (matches PgTune's default here); consider enabling only for a dedicated reporting/analytics workload or database.
- **Reload only — no restart.**

---

## Connections

### `max_connections` — `500`
The hard ceiling on simultaneous client connections the instance will accept.

**Change it?** This is the parameter most worth pushing back on rather than accepting at face value.
- **Cons of a high value like 500:** each connection reserves some memory overhead regardless of whether it's active, and PostgreSQL's own performance can degrade under very high connection counts due to internal contention — a high `max_connections` is often a symptom of *not* using a connection pooler (PgBouncer), not a genuine requirement.
- **Rule of thumb:** size this to your actual measured peak concurrent connection need, not an arbitrary round number. Prefer a connection pooler (PgBouncer) in front of the database and keep `max_connections` closer to what the database itself needs to serve pooled connections — often far lower than 500 in practice.
- **Restart required.**

---

## Quick-Reference: Restart vs. Reload

| Requires restart | Reload only (`pg_reload_conf()` / `pg_ctl reload`) |
|---|---|
| `shared_buffers` | `effective_cache_size` |
| `wal_buffers` | `maintenance_work_mem` |
| `huge_pages` | `work_mem` |
| `io_method` | `checkpoint_completion_target` |
| `max_worker_processes` | `min_wal_size` / `max_wal_size` |
| `max_connections` | `wal_compression` |
| | `random_page_cost` |
| | `effective_io_concurrency` |
| | `default_statistics_target` |
| | `max_parallel_workers` / `max_parallel_workers_per_gather` |
| | `max_parallel_maintenance_workers` |
| | `jit` |

---

## The One-Sentence Version for the Session

**PgTune gets you a defensible starting point for memory and parallelism sizing from hardware specs alone — but `work_mem` and `max_connections` are the two values most likely to need real correction once you have actual production load to look at, because both scale multiplicatively with concurrency in a way a one-time hardware-based calculator can't know in advance.**
