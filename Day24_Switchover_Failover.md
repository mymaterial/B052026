# Manual Switchover, Failover & pg_rewind — Session Notes

Building directly on the streaming replication setup from the previous session, this session covers manual switchover and failover between a primary and standby: the `wal_log_hints` prerequisite, the exact 3-step switchover sequence, what `pg_rewind` actually does and why it's needed even after a "clean" stop, and how `.history` files let PostgreSQL track timeline branch points. Several live troubleshooting moments (including a real `pg_rewind` failure and an instructor mistake) are captured because they illustrate the mechanics better than a clean run would.

---
## Table of Contents

- [1. wal_log_hints — Required Before Any Switchover](#1-wal_log_hints--required-before-any-switchover)
- [2. Failover vs Switchover](#2-failover-vs-switchover)
- [3. The Three Steps of a Switchover](#3-the-three-steps-of-a-switchover)
- [4. pg_ctl promote](#4-pg_ctl-promote)
- [5. pg_rewind — What It Actually Does](#5-pg_rewind--what-it-actually-does)
- [6. Troubleshooting a Real pg_rewind Failure](#6-troubleshooting-a-real-pg_rewind-failure)
- [7. The .history File and Timelines](#7-the-history-file-and-timelines)
- [8. Full Switchover Checklist](#8-full-switchover-checklist)
- [9. Questions Discussed in This Session](#9-questions-discussed-in-this-session)

---

## 1. wal_log_hints — Required Before Any Switchover

```ini
# postgresql.conf — on the primary (and, if already a standby, there too)
wal_log_hints = on
```

`wal_log_hints` adds extra information to WAL records that `pg_rewind` depends on. **This must be set before you ever need to do a switchover or failover** — it's not something you can retroactively enable at the moment you need `pg_rewind` to work; the WAL generated *before* it was turned on won't have the information `pg_rewind` needs. Practical guidance: enable it on every primary as a matter of course, regardless of whether a standby exists yet.

This parameter requires a restart to take effect (confirmed live — the demo restarted the instance immediately after setting it).

---

## 2. Failover vs Switchover

- **Failover:** the primary is gone (crashed, unreachable, destroyed) and you simply promote the standby to become the new primary. You don't care what happens to the old primary — it might come back, it might not.
- **Switchover:** a *planned* role swap — both machines are healthy, and you deliberately make the standby the new primary while turning the old primary into the new standby, with no data loss.

**In PostgreSQL terms: a switchover is failover, plus `pg_rewind`, plus re-establishing replication in the reverse direction.** Failover alone is the easy part (one command); switchover is failover followed by cleanup and re-pointing.

---

## 3. The Three Steps of a Switchover

**Step 1 — Cut the connection between primary and standby, and promote the standby.**

In older PostgreSQL versions (9.x/10), this used to require manually deleting the standby's `standby.signal`/`recovery.conf`. **In current versions, `pg_ctl promote` handles this automatically** — no manual file deletion needed.

```bash
/usr/pgsql-18/bin/pg_ctl promote -D /u01/pgsql18   # run on the standby (Lab02)
```

Once promoted, the two machines behave **completely independently** — each can accept its own writes, and neither sees the other's changes. Any writes the *old primary* accepts after this point are exactly the extra transactions that Step 2 needs to discard.

**Step 2 — Discard any extra changes the old primary accepted after the cutover.**

```bash
pg_ctl stop -D /u01/pgsql18       # stop the old primary first
pg_rewind --target-pgdata=/u01/pgsql18 --source-server="host=192.168.108.132 ..."
```

`pg_rewind` rewinds the old primary's data directory back to the point where it diverged from the new primary (Lab02), discarding whatever extra writes it accepted independently after the cutover. See Section 5 for how it determines that divergence point.

**Step 3 — Re-point the old primary as a standby of the new primary.**

```bash
touch /u01/pgsql18/standby.signal
```
```ini
# postgresql.conf on the old primary
restore_command = 'scp postgres@192.168.108.132:/u01/archive_logs/%f %p'
primary_conninfo = 'host=192.168.108.132 user=postgres password=...'
```
```bash
pg_ctl start -D /u01/pgsql18
```

**Verified live:** after all three steps, writes on the new primary (Lab02) correctly appeared on the old primary (now standby, Lab01) again — full role reversal completed with no data loss.

---

## 4. pg_ctl promote

Promoting a standby is a single command:

```bash
pg_ctl promote -D /u01/pgsql18
```

After promotion, that machine immediately starts accepting writes independently of whatever it was previously following — confirmed live by inserting a record on the newly promoted machine and confirming the write did **not** appear on the old primary (because the connection had already been cut).

---

## 5. pg_rewind — What It Actually Does

Even after a *clean* switchover (application stopped, cron jobs stopped, a checkpoint issued, standby fully caught up before promoting), **`pg_rewind` is still required** — there's no scenario in a switchover where it can be safely skipped. Even a "clean" stop can leave behind internal changes (background process activity, checkpoint records, etc.) that were never sent to the standby before promotion; `pg_rewind` is what guarantees the old primary is byte-for-byte reconcilable with the new primary's timeline before it's turned into a standby.

**What it needs to do its job:** `pg_rewind` needs access to the WAL files covering the period it's rewinding through — either still present in the target's own `pg_wal` directory, or retrievable from the archive location. If a needed WAL file has already been removed from both locations, `pg_rewind` fails outright with an error naming the missing file, and that file has to be manually copied back into `pg_wal` from wherever it's still available (e.g. the archive location) before retrying.

---

## 6. Troubleshooting a Real pg_rewind Failure

A `pg_rewind` failure was deliberately triggered (by removing WAL files from the target's `pg_wal` directory before attempting the rewind) specifically so the class would see and understand the failure mode, rather than only seeing a clean success:

```
pg_rewind: error: could not open file "...": No such file or directory
```

**Fix demonstrated:** manually copy the specific missing WAL file(s) from the archive location back into the target's `pg_wal` directory, then retry `pg_rewind`.

**A separate, unplanned mishap during the same demo:** while trying to remove just the `backup_label` file from a data directory, a typo resulted in an `rm -rf *` that deleted the entire data directory instead. This was called out explicitly as a real mistake, not a demonstrated technique — the recovery was to reinitialize the cluster from scratch (`initdb`, reconfigure `pg_hba.conf`/`postgresql.conf`, re-run `pgbench` to regenerate test data) rather than attempting to salvage a wiped data directory. **Practical lesson:** always double-check a destructive command's exact target before running it, especially when working quickly in a live troubleshooting scenario — a single missing character in an `rm` command can be catastrophic.

---

## 7. The .history File and Timelines

When a standby is promoted, PostgreSQL writes a **`.history` file** recording exactly where the timeline diverged — which WAL position the promotion happened at.

**How `pg_rewind` uses it:** given a source machine's data directory, `pg_rewind` reads that source's `.history` file to determine the exact WAL position where the two machines' timelines split, then rewinds the target back to that shared point before the target can safely follow the source's (new) timeline going forward.

**Timelines can branch more than once** — each promotion creates a new timeline ID and a new `.history` entry, and in principle you can have many branches recorded over a cluster's lifetime, each traceable back to its fork point via these history records.

**Analogy to Oracle's snapshot standby**, raised during Q&A: opening a standby in read-write mode temporarily (to test something against production-like data) and later "reverting" it is conceptually similar to what promoting a standby and then `pg_rewind`-ing it back does in PostgreSQL — the standby branches into its own timeline the moment it starts accepting writes, and `pg_rewind` (using the `.history` file) is what lets you discard that branch and rejoin the original timeline afterward.

---

## 8. Full Switchover Checklist

Consolidated from the session's step-by-step walkthrough:

1. **`wal_log_hints = on`** on the primary, set from the start — not something to add reactively (Section 1).
2. **Stop all applications, cron jobs, and any automated/scheduled activity** connecting to the primary — nothing should be writing during the switchover window.
3. **Issue a `CHECKPOINT`** on the primary.
4. **Confirm the standby is fully caught up** with the primary before proceeding.
5. **Promote the standby** (`pg_ctl promote`) — Step 1 of the switchover (Section 3).
6. **Run `pg_rewind`** against the old primary, even though everything was stopped cleanly — Step 2 (Section 3, Section 5) — this is not optional, ever.
7. **Reconfigure the old primary as a standby** of the new primary (`standby.signal`, `primary_conninfo`, `restore_command`) — Step 3 (Section 3).

---

## 9. Questions Discussed in This Session

**Q1. If an instance has multiple databases, does `pg_basebackup`-based replication replicate all of them, or can a single specific database be replicated on its own — similar to Oracle 19c's per-PDB replication?**

Physical replication (streaming or archive-based, as covered in this and the previous session) always replicates the entire instance — every database in the cluster, together. Replicating a single database on its own requires **logical replication** instead, which is conceptually similar to Oracle's logical replication. However, the recommended fix for "I only want to fail over one application's data" isn't logical replication — it's **not putting many unrelated applications in the same instance in the first place**. Guidance given: hardly 2–3 applications per instance (whether as separate databases or separate schemas within one database) — never something like 20 databases sharing one instance, because then a single backup, restore, or switchover/failover event affects every one of those applications together, which is a design smell independent of which replication technology is used.

---

**Q2. Is there a PostgreSQL equivalent to Oracle's snapshot standby, where you can temporarily open a standby read-write for testing and then revert it back to following the primary?**

Not as a single named feature, but the same outcome is achievable using the mechanics already covered: promote the standby (it starts its own timeline and can now accept writes), test against it, and then use `pg_rewind` against the primary's `.history` file to discard that branch and rejoin it to the original timeline as a standby again. It's a manual composition of `pg_ctl promote` + `pg_rewind` + reconfiguring `standby.signal`, rather than a dedicated single command.

---

**Q3. Is `pg_rewind` ever safe to skip if you're certain the switchover was completely clean (application stopped, checkpoint issued, standby fully caught up)?**

No — the guidance given was unambiguous: always run `pg_rewind` as part of a switchover, regardless of how clean the shutdown appeared. Internal system-level changes can occur independently of application activity, and there's no reliable way to confirm their absence without doing the rewind — treat it as a mandatory step, not a conditional one.
