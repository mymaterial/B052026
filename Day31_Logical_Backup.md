# PgBackRest & Logical Replication — Session Notes

A hands-on tour of two topics deliberately kept brief per the instructor ("these are all small subjects... 10-15 minutes each"): setting up **PgBackRest** as a centralized backup solution (the "stanza" concept, S3 shipping, backup-from-standby considerations) and a full working **logical replication** setup (publication/subscription), closing with a frank discussion of why logical replication is *not* actually used for major-version upgrades in practice, despite being technically possible.

---
## Table of Contents

- [1. PgBackRest — the Centralized Backup Concept](#1-pgbackrest--the-centralized-backup-concept)
- [2. Stanzas — Naming Each Server You Back Up](#2-stanzas--naming-each-server-you-back-up)
- [3. Configuring PgBackRest](#3-configuring-pgbackrest)
- [4. Taking Full and Incremental Backups](#4-taking-full-and-incremental-backups)
- [5. Restoring](#5-restoring)
- [6. Shipping Backups to S3](#6-shipping-backups-to-s3)
- [7. Backing Up From a Standby](#7-backing-up-from-a-standby)
- [8. What PgBackRest Adds Over Plain pg_basebackup](#8-what-pgbackrest-adds-over-plain-pg_basebackup)
- [9. Setting Up Logical Replication](#9-setting-up-logical-replication)
- [10. Why Logical Replication Isn't Actually Used for Upgrades](#10-why-logical-replication-isnt-actually-used-for-upgrades)
- [11. What Logical Replication Is Actually Used For](#11-what-logical-replication-is-actually-used-for)
- [12. Questions Discussed in This Session](#12-questions-discussed-in-this-session)

---

## 1. PgBackRest — the Centralized Backup Concept

**Direct comparison for anyone with an Oracle background:** PgBackRest's role is conceptually equivalent to Oracle's **catalog/recovery catalog server** — a dedicated, centralized location that holds backup copies pulled from multiple production databases, rather than each database managing its own local backups independently.

```
[Lab01 production]  ─┐
[Lab02 production]  ─┼──► [Backup server: PgBackRest]
[Lab03 production]  ─┘
```

**Clarified directly, in response to a question distinguishing this from a true Oracle catalog:** unlike Oracle's catalog (which stores only *metadata about* backups, pointing at where the actual backup files live elsewhere), PgBackRest's backup server genuinely **stores the physical backup files themselves**, centralized in one place — closer to "catalog + actual storage combined" than a pure metadata catalog.

**Best-practice guidance repeated directly:** never rely on a single backup copy in a single location. Standard practice is at least two backup copies in two different locations — e.g. one on the local backup server, one shipped to S3 (or another remote/tape location) — so that losing any one location doesn't mean losing your only backup.

---

## 2. Stanzas — Naming Each Server You Back Up

A **stanza** is PgBackRest's name for a uniquely identified backup target — essentially an alias representing one specific PostgreSQL server/cluster to be backed up.

```ini
# pgbackrest.conf, on the backup server
[lab01]
pg1-path=/u01/pgsql18
pg1-host=192.168.108.131

[lab03]
pg1-path=/u01/pgsql18
pg1-host=192.168.108.133
```

**Confirmed live:** adding a new server to back up is as simple as adding a new stanza block (a few lines: host, data directory path, port) — demonstrated live adding a `lab03` stanza in under a minute.

---

## 3. Configuring PgBackRest

**Prerequisites, consistent with earlier sessions' patterns:** passwordless SSH connectivity between the production server and the backup server (in both directions — the backup server needs to reach production to pull data; production needs to be told where its backup server is via `archive_command`).

```ini
# On the PRODUCTION server's postgresql.conf
archive_mode = on
archive_command = 'pgbackrest --stanza=lab01 archive-push %p'
```

```bash
# On the backup server — initialize the stanza
pgbackrest --stanza=lab01 stanza-create
```

**Confirmed live:** running `stanza-create` for the first time performs a checkpoint on the target and archives that point to the backup location — a successful run confirms the connection and configuration are working end-to-end.

---

## 4. Taking Full and Incremental Backups

```bash
pgbackrest --stanza=lab01 backup
```
**First run:** since no prior backup exists, this automatically performs a **full** backup (180MB in the demo).

```bash
pgbackrest --stanza=lab01 backup   # run again
```
**Second run:** automatically performed an **incremental** backup (14.9KB in the demo) — confirmed directly: **PgBackRest defaults to incremental once a full backup already exists**, without needing to specify the backup type explicitly. An explicit `--type=incr` flag exists too, but produces the same result as just re-running the plain command when a full backup is already present.

```bash
pgbackrest --stanza=lab01 info
```
Lists all backups currently held for a stanza — confirmed showing one full and one incremental backup after the two runs above.

---

## 5. Restoring

**Demonstrated live by deliberately wiping the production data directory first** (`rm -rf` on the data directory contents) to simulate data loss, then restoring:

```bash
# run FROM THE PRODUCTION SERVER, not the backup server
pgbackrest --stanza=lab01 restore
```
```bash
pg_ctl start -D /u01/pgsql18
psql -c "SELECT * FROM employee;"   -- data confirmed restored
```

**Confirmed directly, in response to a question about restoring to a different directory:** by default, restore always targets the **original configured data directory path** — restoring to an alternate location isn't a runtime flag choice; it requires **first editing the stanza's configured path** in `pgbackrest.conf`, then running restore, which will place the restored files at whatever path is currently configured.

---

## 6. Shipping Backups to S3

```ini
[global]
repo1-type=s3
repo1-s3-bucket=<your-bucket-name>
repo1-s3-endpoint=<region-endpoint>
repo1-s3-key=<access-key>
repo1-s3-key-secret=<secret-key>
```
Same `backup`/`restore`/`info` commands work identically once configured for an S3-backed repository instead of a local one — PgBackRest handles the transfer to S3 the same way it would to a local backup server.

**Practical context given:** this is especially relevant when PostgreSQL itself is hosted on EC2 — data transfer from EC2 to S3 within the same region is typically fast and often free/low-cost, making S3 a natural, cost-effective secondary (or even primary) backup destination for cloud-hosted instances. **Setup requirement:** create the S3 bucket and appropriate IAM permissions first, then configure PgBackRest with those bucket/credential details.

---

## 7. Backing Up From a Standby

**A detailed question worked through:** in a 3-node Patroni cluster where a switchover has occurred (the node originally configured as the PgBackRest backup source is no longer the primary), does PgBackRest need special handling?

**Answer, stated directly:** **whether the backup source is a primary or a standby has nothing to do with PgBackRest's own logic** — it doesn't distinguish between the two. All that's actually required is that the connecting user has the necessary read/replication-level access to pull data from whichever node is configured as the source. Internally, PgBackRest's backup operation is effectively running the equivalent of `pg_basebackup` against whichever node is configured — role awareness (primary vs. standby) isn't something PgBackRest itself manages; that's a separate, manual configuration decision (which stanza points at which node).

**Practical implication:** in an HA cluster where the primary can change via failover/switchover, keeping PgBackRest correctly pointed at "whichever node is currently primary" requires either manually updating the stanza configuration after a role change, or building that update into your failover automation — it isn't handled automatically by PgBackRest itself.

---

## 8. What PgBackRest Adds Over Plain pg_basebackup

**Confirmed directly: internally, PgBackRest's actual backup mechanism is `pg_basebackup`** — there's no different underlying technology. What PgBackRest adds on top:

| Feature | Plain `pg_basebackup` | PgBackRest |
|---|---|---|
| Parallelism (multiple CPUs for backup speed) | No — single-threaded | Yes — configurable (`process-max`) |
| Retention policy (automatic obsolete-backup cleanup) | No — manual | Yes — configurable (e.g. keep last 2 full backups) |
| S3/cloud shipping | No — manual scripting required | Built-in |
| Restore a single database out of a multi-database cluster | No | **Also no** — explicitly confirmed this limitation carries over; PgBackRest restores are still whole-cluster or nothing, matching the instance-wide backup constraint from the earlier session |

**Direct Oracle comparison for the retention behavior:** PgBackRest's retention policy is described as functioning similarly to Oracle's "obsolete" backup marking/cleanup — configure how many full backups to retain, and older ones beyond that count are automatically cleaned up.

---

## 9. Setting Up Logical Replication

**Step 1 — enable logical WAL decoding on the source (primary):**
```ini
# postgresql.conf, on the primary
wal_level = logical
```
Restart required (this is the same `wal_level` parameter from earlier replication sessions, now set one level higher than plain streaming replication needs).

**Step 2 — copy the schema (metadata only) from source to target, since logical replication only replicates data, not schema:**
```bash
pg_dump -U <user> -d <source_db> -s > meta.sql
```
```bash
psql -U <user> -d <target_db> -f meta.sql
```
**Confirmed: `-s` (schema-only) is the key flag** — logical replication assumes the target already has matching table structures; it does not create them for you.

**Step 3 — on the primary (publisher side), create a publication** naming which tables to replicate:
```sql
CREATE PUBLICATION my_publication FOR ALL TABLES;
-- or, for a specific table only:
CREATE PUBLICATION my_publication FOR TABLE employee;
```

**Step 4 — on the target (subscriber side), create a subscription** pointing back at the publisher:
```sql
CREATE SUBSCRIPTION my_subscription
CONNECTION 'host=192.168.108.154 dbname=A'
PUBLICATION my_publication;
```

**Confirmed live:** creating the subscription automatically creates a **replication slot** on the publisher and triggers an initial data sync (functionally similar to a `pg_dump`/`pg_restore` cycle under the hood) — **sync time scales with data volume**, same caveat as any bulk data-copy operation.

```sql
SELECT * FROM pg_stat_subscription;   -- check sync status, similar in spirit to checking physical replication lag
```

**Confirmed live end-to-end:** an `INSERT` and `UPDATE` on the primary appeared correctly on the logical replica after the initial sync completed.

---

## 10. Why Logical Replication Isn't Actually Used for Upgrades

**Explicitly and directly stated, despite it being technically demonstrable:** logical replication *can* replicate across different major versions (e.g. 17 → 18), which theoretically enables a near-zero-downtime upgrade pattern (sync via logical replication, cut the application over to the new version, done). **But in real-world practice, this is essentially never actually used** — the instructor was blunt: *"nobody uses that... for an interview, you can mention it, but in reality, `pg_upgrade` with link mode [4-5 minutes of downtime] is what everyone actually uses."*

**Concrete reasons given for why teams avoid it:**
1. **A known, longstanding sequence-value replication problem** — logical replication historically does not automatically carry sequence current-values across correctly, requiring manual intervention to align sequence state between primary and replica after cutover. (Noted as reportedly addressed in version 19 — "need to see" if that holds up in practice.)
2. **Tables without a primary key cannot be logically replicated at all.** Real production schemas commonly have some tables without primary keys, or with duplicate data that would need cleanup before a primary key could even be added — creating substantial, unplanned remediation work just to make logical replication viable.
3. **The net result, from the instructor's direct experience:** teams that tried this approach ran into enough friction that developers pushed back and the approach was abandoned in favor of the much simpler `pg_upgrade` (link mode) path with its short, predictable downtime window.

**"Zero-downtime" and "blue-green deployment" framed skeptically:** described directly as "just for the namesake" in practice — the actual tradeoffs (sequence handling, primary-key requirements, general complexity) make the supposedly-zero-downtime path more operationally risky than a brief, well-understood `pg_upgrade` window for most real teams.

---

## 11. What Logical Replication Is Actually Used For

**The two genuine, common use cases, stated directly:**

1. **Reporting workloads** — offloading read-heavy reporting queries onto a logical replica so they don't add load to the primary, without needing the entire instance replicated (physical replication would copy the whole instance regardless of database count; logical replication can be scoped down to exactly the tables needed for reporting).
2. **Granular, selective replication** — logical replication can be configured at the **table level**, or even the **partition level** — described directly as being like Oracle's PDB (pluggable database) concept in flexibility: "I want only these 4 tables replicated" is achievable with logical replication, whereas physical replication is all-or-nothing at the instance level.

**A practical requirement reinforced for table-level logical replication specifically:** any table you intend to logically replicate needs a genuine primary key — this is a real, non-negotiable prerequisite check to do before attempting to add a table to a publication.

---

## 12. Questions Discussed in This Session

**Q1. Does PgBackRest care whether it's backing up from a primary or a standby node?**

No — PgBackRest itself has no special-case logic distinguishing primary from standby; it just needs read/replication access to whichever node is configured as its source. In an HA cluster where the primary can change via failover, keeping the backup configuration pointed at the current primary is a manual (or automation-driven) responsibility, not something PgBackRest manages on its own.

---

**Q2. Can you restore a PgBackRest backup to a different directory than the one it was originally taken from?**

Not as a runtime restore-command flag — the target path is determined by the stanza's currently configured data-directory path in `pgbackrest.conf`. To restore elsewhere, you first edit that configuration to point at the new desired path, then run the restore.

---

**Q3. Is logical replication a good choice for major version upgrades, given it can technically bridge different PostgreSQL versions?**

Technically yes, but practically no — the real-world guidance given was to avoid it for upgrades specifically, due to historical sequence-value handling problems and the hard requirement that every replicated table have a primary key (often not true of real production schemas without cleanup work). `pg_upgrade` with link mode, with its short and predictable downtime window, is what's actually used in practice; logical-replication-based "zero-downtime" upgrades are more something to mention as theoretical knowledge in an interview than something teams actually deploy.

---

**Q4. What are the actual, common reasons people set up logical replication in production?**

Reporting workload offloading (so reporting queries don't burden the primary) and granular, table/partition-level selective replication (replicate only specific tables rather than an entire instance) — not major version upgrades, which is the common misconception addressed directly in this session.
