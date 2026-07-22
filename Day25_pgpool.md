# Query Performance Triage, PGPool & Patroni Theory — Session Notes

This session opens with a real production troubleshooting discussion (a pre-prod environment seeing 2ms queries balloon to 100ms) before moving into a full hands-on `pgpool-II` setup — connection caching, read/write load balancing, and its (weak) high-availability story — and closes with the conceptual architecture behind Patroni-based automatic failover, ahead of hands-on Patroni configuration in the next session.

---
## Table of Contents

- [1. Troubleshooting a Real Latency Regression](#1-troubleshooting-a-real-latency-regression)
- [2. What PGPool Actually Is](#2-what-pgpool-actually-is)
- [3. Installing and Configuring PGPool](#3-installing-and-configuring-pgpool)
- [4. Verifying Load Distribution with pgbench](#4-verifying-load-distribution-with-pgbench)
- [5. PGPool's Three Capabilities, and Its HA Limitation](#5-pgpools-three-capabilities-and-its-ha-limitation)
- [6. Pinning Specific Traffic to the Standby](#6-pinning-specific-traffic-to-the-standby)
- [7. Patroni — Conceptual Architecture](#7-patroni--conceptual-architecture)
- [8. Passwordless SSH and Patroni](#8-passwordless-ssh-and-patroni)
- [9. Custom Ports Are Fine Everywhere](#9-custom-ports-are-fine-everywhere)
- [10. PgBouncer — Started, To Be Continued](#10-pgbouncer--started-to-be-continued)
- [11. Questions Discussed in This Session](#11-questions-discussed-in-this-session)

---

## 1. Troubleshooting a Real Latency Regression

A student described a live pre-prod issue: queries that used to run in ~2ms were now taking ~100ms, alongside connection-timeout complaints and high CPU/temp-file usage. Working through it as a structured triage:

- **Load-test numbers and real production numbers are not directly comparable — don't anchor to the load test figure.** A load test reporting "2ms" doesn't mean production traffic will behave the same way; real application traffic has different concurrency patterns, data volumes, and cache states than a synthetic benchmark. Treating a load-test result as the expected baseline for production is a common mistake.
- **Use `pg_stat_statements` as the primary source of truth** for on-premise environments; for cloud-managed databases, check whether advanced/granular monitoring (e.g. AWS RDS Performance Insights, or equivalent CloudWatch-level detail) is actually enabled — many environments have it turned off by default, which limits what you can diagnose.
- **The systematic checklist for "why did this get slower," worked through explicitly:** has there been a query-level change, a schema-level change (e.g. `ALTER TABLE`), a table-level data change (significant growth or a shift in data distribution), a block/page-level change, or a PostgreSQL version change? Any of these can independently cause the query planner to pick a different (worse) execution plan than before.
- **The core diagnostic principle:** don't jump to a fix — first establish exactly *what changed*, one category at a time, before proposing a solution. "It got slower" isn't itself actionable; "the query plan changed because table X grew past a statistics threshold" is.

---

## 2. What PGPool Actually Is

`pgpool-II` is a small piece of middleware software, installed on its own machine (or the same machine as one of the database nodes), that sits between the application and the PostgreSQL primary/standby pair. The application only ever talks to PGPool's IP address; PGPool decides where each request actually goes.

**Its core behavior:**
- **Read-write requests** are always routed to the primary.
- **Read-only requests** are load-balanced between the primary and the standby.
- The application's connection string changes only in **IP address** — username, password, port (still `5432`) stay the same, pointed at PGPool's IP instead of the primary's IP directly.

**When PGPool is actually worth adding:** when the primary is consistently running hot (e.g. 70–80% CPU) and offloading read-only traffic to the standby would meaningfully help. If the primary is comfortably under load (e.g. ~50% CPU), there's no real benefit to adding PGPool — it's not a default addition to every setup.

**A related but different ask sometimes comes from management:** "since we have PGPool now, can we shrink the primary's compute and rely on load balancing to make up the difference?" — a legitimate question to be asked *of you*, not something to preemptively suggest yourself; sizing changes like that need their own justification, not just "well, we have PGPool now."

---

## 3. Installing and Configuring PGPool

```bash
# on the third machine (Lab03), acting as the PGPool node
dnf install pgpool-II
```

```bash
cp /etc/pgpool-II/pgpool.conf.sample-stream /etc/pgpool-II/pgpool.conf
vi /etc/pgpool-II/pgpool.conf
```

**Key settings edited:**
```ini
listen_addresses = '*'

# backend definitions — one per PostgreSQL node PGPool should know about
backend_hostname0 = '192.168.108.131'   # primary
backend_data_directory0 = '/u01/pgsql18'
backend_hostname1 = '192.168.108.132'   # standby
backend_data_directory1 = '/u01/pgsql18'
```

```bash
pg_ctl start   # (PGPool's own start command, per its installed binaries)
```

**Verifying it's running:**
```bash
export PGPASSWORD=postgres
psql -h <pgpool_host> -p 9999 -U postgres -c "show pool_nodes;"
```

PGPool listens on port **9999** by default (confirmed configurable — see Section 9). A working `show pool_nodes` output shows both backend IPs as up, one marked primary and the other standby.

---

## 4. Verifying Load Distribution with pgbench

**Read-write traffic — should go entirely to the primary:**
```bash
pgbench -c 10 -t 10 -p 9999 -h <pgpool_host> postgres
```
Result: 100 transactions on the primary, 0 on the standby — confirming read-write requests never get load-balanced.

**Read-only traffic — should be distributed:**
```bash
pgbench -S -c 10 -t 10 -p 9999 -h <pgpool_host> postgres
```
First run: roughly 70/30 split between primary and standby. A second run with a different client seed distributed closer to 50/50. **Load balancing is approximate, not a guaranteed exact split** — the point demonstrated was that both nodes genuinely receive read traffic, not that the split is mathematically precise every time.

---

## 5. PGPool's Three Capabilities, and Its HA Limitation

| Capability | PGPool | PgBouncer | Patroni |
|---|---|---|---|
| Connection caching (retain/reuse established connections) | Yes | Yes — this is PgBouncer's core purpose | No (not its job) |
| Read/write load balancing | Yes | **No** | No (not its job directly — see Section 7) |
| Automatic switchover/failover (HA) | Yes, but **not robust** | No | Yes — this is Patroni's core purpose |

**Application-level connection pooling** (e.g. HikariCP, covered in an earlier session) is the equivalent of connection caching done at the application layer; PGPool is the same idea done at the database-tier layer.

**PGPool's built-in failover mechanism is a simple health-check script** — periodically check whether the primary is reachable; if not, run `pg_ctl promote` on the standby. This was demonstrated as literally a shell script performing this check in a loop, not a sophisticated consensus-based system.

**Explicitly flagged as not recommended for production HA reliance:** PGPool's failover has known reliability problems in practice, and the class was told directly not to depend on it for genuine high availability — that's what Patroni is for (Section 7). PGPool can technically do all three things in the table above, but only connection caching and load balancing are the parts worth trusting it for; treat its failover capability as a fallback at best, not a primary HA mechanism.

---

## 6. Pinning Specific Traffic to the Standby

**Scenario raised:** a heavy reporting job runs from one specific application/machine, and the team wants to force *only that workload* onto the standby, without relying on PGPool's automatic (and non-configurable-by-request) load balancing.

**PGPool cannot be told "always route this specific request to the standby"** — its distribution logic is automatic and not something you can override per-request through PGPool itself.

**The actual solution: bypass PGPool for that specific workload** and have the reporting application connect directly to the standby's physical IP address, using its own overridden connection string instead of the shared PGPool connection string used elsewhere.

**The failover risk this introduces:** if that standby is later promoted (during a failover), the reporting job's hardcoded IP now points at what has become the new primary — the report would still technically run (since it's mostly `SELECT`), but the isolation intent (keep this workload off the primary) is now broken, and someone has to manually update the reporting application's target IP.

**A more resilient pattern discussed:** have the reporting script check `pg_is_in_recovery()` before running — this returns `true` on a standby and `false` on a primary. The script's logic can branch on this: if it detects it's *not* talking to a standby, either abort (per the simplest example shown) or dynamically pick the correct standby IP as an argument, rather than assuming a hardcoded IP is always correct.

```sql
SELECT pg_is_in_recovery();  -- true on a standby, false on a primary
```

---

## 7. Patroni — Conceptual Architecture

Patroni provides genuinely automated failover, in contrast to PGPool's simple script-based approach.

```
[Postgres primary, 131] --- agent ---\
                                        > etcd (decision-making layer)
[Postgres standby, 132] --- agent ---/
```

**How it works:**
1. A small **agent process** runs alongside PostgreSQL on each node.
2. Each agent continuously sends health-check information to **etcd** (a distributed key-value store acting as the cluster's shared source of truth), roughly every few seconds, 24×7.
3. **If the primary's agent stops reporting in** (default example given: after ~30 seconds of silence), etcd instructs the surviving node's agent to `pg_ctl promote` itself.
4. This is what makes the failover **automatic** — no human has to notice the primary is down and manually run `pg_ctl promote`.

**Why etcd itself needs to be a cluster, not a single machine:** if etcd is just one machine and *that* machine goes down, there's no decision-maker left at all, and automatic failover breaks entirely — a single point of failure defeats the purpose. The standard pattern is a **3-node etcd cluster**, using distributed consensus so the cluster keeps functioning correctly even if one etcd node is lost.

**Minimum footprint discussed for this teaching example:** 3 machines running etcd, plus the primary and standby PostgreSQL nodes — **5 machines total** for a genuinely automated HA setup (in this particular architecture, with etcd on separate dedicated machines; some real-world deployments instead co-locate etcd with the PostgreSQL nodes themselves to reduce the machine count — that variant wasn't covered in this session).

**Load balancing on top of a Patroni cluster** requires an additional piece — **HAProxy** — layered on top of etcd, distributing read-only traffic **between the standbys** (in a multi-standby cluster) rather than between primary and standby the way PGPool does. Patroni's job is failover; HAProxy's job, when added, is the load-balancing piece PGPool would otherwise provide.

---

## 8. Passwordless SSH and Patroni

**Clarified directly:** Patroni's normal, ongoing operation does **not** require passwordless SSH between the cluster's nodes — the agents communicate through etcd, not directly with each other over SSH.

**Where passwordless SSH does matter:** only during the *initial automated setup* of a multi-node cluster, if you're using a script run from a single control/jump machine to configure all the nodes remotely in one pass. Once that one-time setup automation is complete, **the passwordless SSH access should be revoked** — leaving it in place afterward is an unnecessary, avoidable security exposure, not something Patroni needs to keep running.

---

## 9. Custom Ports Are Fine Everywhere

**Question raised:** if an environment has restrictions preventing use of default ports, can PGPool (`9999` default), PgBouncer, and HAProxy all be configured to use custom ports instead?

**Yes** — confirmed directly in each tool's own configuration file (`pgpool.conf`'s `port` setting shown live as the example) — none of these tools' default ports are hardcoded or mandatory; they're ordinary configuration values like any other.

---

## 10. PgBouncer — Started, To Be Continued

Installation and initial config file location were covered as a lead-in to a fuller PgBouncer session next time:

```bash
dnf install pgbouncer
vi /etc/pgbouncer/pgbouncer.ini
```

The `port` setting in `pgbouncer.ini` was pointed out (same principle as Section 9 — fully configurable), but detailed configuration and tuning of PgBouncer itself was explicitly deferred to the next session.

---

## 11. Questions Discussed in This Session

**Q1. Does PGPool combine multiple client connections into a single backend connection, reducing memory usage that way?**

No — that's not what's happening. Each incoming connection PGPool distributes still becomes its own individual connection on whichever backend (primary or standby) it's routed to; there's no merging of connections into one. The memory benefit of load balancing comes from **splitting** the total connection count across two machines instead of concentrating all of it on one — e.g. 100 connections at ~2MB each is 200MB of memory pressure on a single machine, but split 50/50 across primary and standby, each machine only carries about 100MB of that load.

---

**Q2. Can PGPool be configured to always route a specific application's traffic to the standby, bypassing its normal automatic load-balancing?**

Not through PGPool's own configuration — its distribution logic is automatic and isn't designed for per-request manual overrides. The practical workaround is to have that specific application connect directly to the standby's IP address with its own overridden connection string, bypassing PGPool entirely for that workload (see Section 6) — with the caveat that this hardcoded IP becomes stale if that standby is later promoted during a failover.

---

**Q3. If PGPool can technically do connection caching, load balancing, *and* failover, why not just rely on PGPool alone instead of also using Patroni?**

Because PGPool's failover mechanism (Section 5) is a simple health-check-and-promote script, not a robust, production-grade HA system — it's explicitly not recommended to depend on for real high availability. The practical pattern is: use Patroni specifically for automatic failover (Section 7), and optionally layer PGPool or PgBouncer/HAProxy on top for connection caching and load balancing, rather than expecting one tool to reliably handle all three responsibilities.

---

**Q4. Does Patroni require passwordless SSH connectivity between the primary and standby nodes to function?**

No, not for its ongoing operation — nodes communicate their health status through etcd, not directly with each other. Passwordless SSH is only useful (and only needed) as a convenience during the initial cluster setup if you're scripting the configuration of multiple machines from one control node, and that access should be revoked once setup is complete rather than left in place indefinitely.
