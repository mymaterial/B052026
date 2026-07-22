# PostgreSQL Versioning Policy & pg_upgrade — Session Notes

This session covers PostgreSQL's version-numbering and release-support policy, a live minor-version upgrade (binary-only, no downtime risk to data), and a full major-version upgrade using `pg_upgrade` in both copy mode and link mode — including the real trade-offs between the two that determine which one you'd actually choose in production.

---
## Table of Contents

- [1. PostgreSQL Version Numbering](#1-postgresql-version-numbering)
- [2. Release Cadence and the 5-Version Support Window](#2-release-cadence-and-the-5-version-support-window)
- [3. When to Adopt a New Major Version](#3-when-to-adopt-a-new-major-version)
- [4. Any Version to Any Version — No Intermediate Hops](#4-any-version-to-any-version--no-intermediate-hops)
- [5. Minor Version Upgrade](#5-minor-version-upgrade)
- [6. Major Version Upgrade — Preparing pg_upgrade](#6-major-version-upgrade--preparing-pg_upgrade)
- [7. Copy Mode](#7-copy-mode)
- [8. Link Mode](#8-link-mode)
- [9. Choosing Copy vs. Link Mode](#9-choosing-copy-vs-link-mode)
- [10. Link Mode's Real Risk](#10-link-modes-real-risk)
- [11. Post-Upgrade Steps](#11-post-upgrade-steps)
- [12. What's Left for pg_upgrade](#12-whats-left-for-pg_upgrade)
- [13. Questions Discussed in This Session](#13-questions-discussed-in-this-session)

---

## 1. PostgreSQL Version Numbering

PostgreSQL uses a simple two-part version number: **major.minor** — e.g. `18.4`, `17.7`. There's no third "patch" tier and no intermediate version concept beyond that.

- **Minor version upgrade:** e.g. `17.6` → `17.10` — same major version, later minor number.
- **Major version upgrade:** e.g. `17.x` → `18.x` — a new major version entirely.

---

## 2. Release Cadence and the 5-Version Support Window

- A new **major version** is released roughly once a year, typically around the **third Thursday of the release quarter** (historically Q4/around September–November).
- **Minor version releases** for all currently supported major versions happen together, roughly quarterly, alongside security/bug fixes.
- **PostgreSQL maintains the latest 5 major versions at any given time.** When a new major version ships, the oldest supported version drops out of support. Concretely: with 18 as current, the community supports 18, 17, 16, 15, and 14 — version 13 and earlier no longer receive fixes or community support for reported issues.

**Practical rule stated directly:** your production database should always be on one of the latest 5 supported major versions. If it isn't, you're running an unsupported version and won't get help from the community for problems you hit.

---

## 3. When to Adopt a New Major Version

**Don't jump onto a brand-new major version (`x.0`) the moment it's released.** Recommended practice: wait roughly **two quarters**, until that major version has reached its `.2` (or `.3`) minor release, before planning a production upgrade onto it.

**Contrasted explicitly with the Oracle "N-1" convention** (staying one major version behind the latest, as a matter of policy): PostgreSQL doesn't need that kind of blanket lag. You can adopt the *current* latest major version — you just shouldn't adopt it on day one. Example given: version 19 releases in November; the practical earliest adoption point is once 19.2 ships, roughly two quarters later, not immediately in November.

---

## 4. Any Version to Any Version — No Intermediate Hops

**Explicitly confirmed:** there is no requirement to upgrade through intermediate major versions. You can go directly from version 13 to 18, or 14 to 18, or 15 to 18 — `pg_upgrade` doesn't require you to pass through 14→15→16→17 along the way. Upgrade from any supported (or even unsupported, though unsupported by community help) version directly to any target version.

---

## 5. Minor Version Upgrade

A minor version upgrade is purely a **binary swap** — it never touches the actual database files, so there's no real data risk involved.

```bash
dnf install postgresql17-server        # installs the latest available 17.x binaries
# or, to pin a specific minor version:
dnf install postgresql17-server-17.9   # installs exactly 17.9
```

**Important nuance demonstrated live:** installing the new binaries doesn't immediately change what a *running* instance reports — `psql` (the client tool) will show the new version right away, but the already-running **server** process keeps reporting its old version until it's restarted:

```bash
psql   # client tool now shows 17.10
# but the running server still reports 17.6 until restarted
pg_ctl restart -D /u01/pgsql17
psql   # now both client and server show 17.10
```

**Recommended cadence:** run this every ~3 months, whenever a new minor release ships — there's no reason to delay a minor upgrade, since it carries essentially none of the risk a major upgrade does.

**Rolling back a minor version** is just as simple — reinstall the specific older version by name and restart:
```bash
dnf install postgresql17-server-17.6
pg_ctl restart -D /u01/pgsql17
```

---

## 6. Major Version Upgrade — Preparing pg_upgrade

Unlike a minor upgrade, a **major** version upgrade does touch the database's on-disk format — this is where real risk and planning enter the picture.

**Step 1 — install the new major version's binaries** (this alone doesn't touch your existing cluster):
```bash
dnf install postgresql18-server
```

**Step 2 — also install matching versions of every third-party extension you use**, for the new major version:
```bash
dnf install postgresql18-contrib
dnf install pg_repack_18   # example: any extension used on 17 needs its 18-compatible build too
```
**This is a real, easy-to-miss prerequisite** — if an extension used on the old version doesn't have an installed build for the new version, `pg_upgrade` will fail its compatibility check.

**Step 3 — initialize a fresh, empty data directory for the new version:**
```bash
usr/pgsql-18/bin/initdb -D /u01/pgsql18
```

**Step 4 — stop the old version's running instance** — `pg_upgrade` requires the source cluster to be shut down:
```bash
pg_ctl stop -D /u01/pgsql17
```

**Step 5 — run `pg_upgrade`:**
```bash
/usr/pgsql-18/bin/pg_upgrade \
  -d /u01/pgsql17 -D /u01/pgsql18 \
  -b /usr/pgsql-17/bin -B /usr/pgsql-18/bin \
  --check
```
`-d`/`-D` are the old/new **data** directories; `-b`/`-B` are the old/new **binary** directories. Running with `--check` first performs a **dry-run compatibility check only** — no changes are made — confirming things like extension availability and cluster compatibility before you commit to the real run.

---

## 7. Copy Mode

Running `pg_upgrade` with no extra mode flag defaults to **copy mode**: it physically copies every data file from the old cluster's data directory into the new one.

```bash
/usr/pgsql-18/bin/pg_upgrade -d /u01/pgsql17 -D /u01/pgsql18 -b /usr/pgsql-17/bin -B /usr/pgsql-18/bin
```

**Confirmed live via disk usage:** after a copy-mode upgrade, the new version's data directory grows to match the old one's size (`du -sh` showed roughly the same size on both, e.g. ~778MB on each side after copying).

**Characteristics:**
- **No backup strictly required beforehand** — if the copy fails partway, the old cluster's data directory is completely untouched; you can simply delete the failed new directory and retry, or just keep running the old cluster.
- **Requires free disk space equal to your database size** — a 4TB database needs 4TB of additional free space at the target location during the copy.
- **Downtime is bounded by raw copy speed** — roughly your disk/storage throughput (commonly 20–90 Mbps in typical setups), applied to your total database size. A 200–300GB database might copy in 5–10 minutes; a multi-terabyte database could mean a much longer downtime window.
- Once the upgrade finishes successfully, the **old cluster's data directory is no longer needed** and can be removed (`pg_upgrade`'s own output explicitly says so).

---

## 8. Link Mode

```bash
/usr/pgsql-18/bin/pg_upgrade -d /u01/pgsql17 -D /u01/pgsql18 -b /usr/pgsql-17/bin -B /usr/pgsql-18/bin -k
```

`-k` (link mode) creates **hard links** to the old cluster's data files instead of copying them — the underlying table/index data physically stays in place on disk, in the old cluster's files; the new cluster's data directory just gets link entries pointing at the same physical storage.

**Confirmed live via disk usage:** after a link-mode upgrade, the new cluster's data directory size was tiny (tens of MB — just control-file and metadata-level content), while the old cluster's directory retained its full original size — because the actual table data was never copied, just linked.

**Characteristics:**
- **Dramatically faster** — downtime is governed by how long it takes the OS to create hard links for every physical file (one link per table/index file, i.e. roughly per OID), not by data volume. Cited benchmark figures (from third-party research, tool/version-dependent, presented as ballpark not guaranteed): ~7 seconds for 128 tables, ~2.5 minutes for 20,000 tables, ~10 minutes for 100,000 tables — **table count drives the time, not database size in GB**.
- **No meaningful extra disk space required** — since data isn't duplicated.
- **A real backup is required beforehand** — because if something goes wrong *while* links are being created, you can end up with **both** the old and new cluster corrupted/unusable simultaneously, since they're sharing the same underlying physical files during that window. The official `pg_upgrade` documentation is explicit about this: if the process aborts *before* linking starts, the old cluster is untouched and can simply be restarted as-is; but once linking is underway, a failure partway through is a genuinely dangerous state, and restoring from backup may be the only safe recovery path.

---

## 9. Choosing Copy vs. Link Mode

Practical decision framework given directly:

| Scenario | Recommendation |
|---|---|
| Client accepts ~10 minutes of downtime, database is a few hundred GB | Copy mode — simpler, no pre-upgrade backup strictly required, and the downtime is genuinely small at that size |
| Database exceeds ~1TB, or downtime tolerance is small | Link mode — copy-mode downtime at that scale becomes unacceptable to most clients nowadays |
| Uncomfortable with link mode's shared-file risk window | Convince the client to accept a longer downtime and use copy mode instead — but this is increasingly hard to sell, since large downtime windows are rarely acceptable in modern production environments |

**Bottom line stated directly:** "you have to pray nothing goes wrong during the linking window" if you choose link mode — it's the fast option, but its failure mode is meaningfully worse than copy mode's, which is exactly why a backup is mandatory before attempting it.

---

## 10. Link Mode's Real Risk

A student's follow-up question clarified an important nuance: it's **not** the table structure or actual row data that gets copied even in link mode — no table content is ever duplicated in link mode, by definition. What takes time (and creates the risk window) is the **operating-system-level work of creating a hard link for every physical data file** (roughly one file per table/index, keyed by OID) — this is pure filesystem work, unrelated to PostgreSQL's own internal logic, which is why it scales with table *count*, not table *size*.

---

## 11. Post-Upgrade Steps

- **Back up your configuration files before upgrading** — copy the old version's `postgresql.conf` and `pg_hba.conf` aside; the new version's default config files need to be edited to match your actual production settings (they don't carry over automatically).
- **If you have a standby, it must be reconfigured after the upgrade** (covered in more depth in a following session) — PostgreSQL has no concept of "upgrade the standby, then roll over to primary" the way some other databases do; using copy mode in particular leaves you no choice but to fully reconfigure the standby from scratch against the newly upgraded primary.
- **Run `ANALYZE` across the entire database after upgrading** — refreshes planner statistics, since a major version upgrade can genuinely change query plans.
- **`ANALYZE` alone isn't guaranteed to prevent a regression** — capture `EXPLAIN` plans for your important/critical queries *before* the upgrade (or rely on an existing observability setup that already captures plan history), so that if performance degrades post-upgrade, you have a concrete "before" baseline to compare against rather than debugging blind.

---

## 12. What's Left for pg_upgrade

Explicitly flagged as continuing in the next session:
- Upgrading a **primary with an existing standby/replica** attached (not just a standalone instance, as demonstrated today).
- **Logical replication** as an alternative upgrade path.
- Re-running the **link mode** demo more carefully.
- A **16 → 17 → 18** chained example, and what changes (if anything) once a newer version like 19 exists — specifically whether link mode's "don't delete the old cluster" caution creates any kind of rollback/rollover consideration worth understanding explicitly.

---

## 13. Questions Discussed in This Session

**Q1. After installing new binaries via `dnf`, why does `psql` immediately show the new version while the running server doesn't?**

`dnf install` only updates the binaries in your installation/bin directory — it doesn't touch or restart the already-running server process. `psql` (a freshly invoked client each time) picks up the newly installed client binary immediately, but the running server keeps using whatever binary it was originally started with, in memory, until you explicitly restart it (`pg_ctl restart`). After a restart, both client and server report the new version.

---

**Q2. If a link-mode `pg_upgrade` fails partway through, is a backup restore always required to recover?**

Not always — it depends on exactly when the failure occurred. If it fails *before* any linking has actually started, the old cluster is untouched and can simply be restarted normally. But once linking is underway, a failure can leave both the old and new cluster in a genuinely unusable state, since the underlying physical files are shared between them during that window — at that point, restoring from a pre-upgrade backup may be the only reliable recovery path. This is exactly why a backup is mandatory before attempting link mode, even though it's not guaranteed to be needed.

---

**Q3. Does the number of tables in a database actually affect link-mode upgrade time because of some database-level metadata cost (e.g. control file size), or something else?**

Something else — it has nothing to do with control file size or any PostgreSQL-internal metadata cost scaling with table count. It's purely the **operating system's** cost of creating one hard link per physical data file (each table/index maps to a physical file, identified by its OID). More tables means more individual link-creation operations at the OS level, which is why link-mode duration scales with table *count* rather than with total data *size* in GB.

---

**Q4. Do you need to upgrade through every intermediate major version (e.g. 14 → 15 → 16 → 17 → 18), or can you jump directly?**

You can jump directly between any two supported major versions — there's no requirement to pass through intermediate versions along the way. `pg_upgrade` supports upgrading straight from, for example, version 13 or 14 all the way to version 18 in a single operation.
