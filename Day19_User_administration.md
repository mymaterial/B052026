# Users, Schemas, search_path & Role-Based Auditing — Reference Guide

Covers PostgreSQL's user/schema model, the `search_path` resolution mechanism, the standard read-only-grant pattern, and a full role-based-access-control setup with per-user statement auditing.

---
## Table of Contents

- [1. The Namespace Hierarchy, Recap](#1-the-namespace-hierarchy-recap)
- [2. Creating a User and an Owned Database](#2-creating-a-user-and-an-owned-database)
- [3. A Schema Named After Its Owner — Why That Matters](#3-a-schema-named-after-its-owner--why-that-matters)
- [4. Creating a Read-Only User — the Three-Step Grant](#4-creating-a-read-only-user--the-three-step-grant)
- [5. GRANT ... ON ALL TABLES Is Not Retroactive](#5-grant--on-all-tables-is-not-retroactive)
- [6. ALTER DEFAULT PRIVILEGES — Covering Future Tables](#6-alter-default-privileges--covering-future-tables)
- [7. Multiple Schemas and search_path](#7-multiple-schemas-and-search_path)
- [8. Same Table Name, Different Schemas — Resolution Order](#8-same-table-name-different-schemas--resolution-order)
- [9. Activity — Role-Based Access Control](#9-activity--role-based-access-control)
- [10. A Likely Copy-Paste Error in the Source Steps](#10-a-likely-copy-paste-error-in-the-source-steps)
- [11. Per-User Statement Auditing](#11-per-user-statement-auditing)
- [12. Decoding log_line_prefix](#12-decoding-log_line_prefix)

---

## 1. The Namespace Hierarchy, Recap

```
Server (cluster / instance)
 └── Database
      └── Schema
           └── Table
```

Everything in this document happens **within one database** (`kfc_db`), showing how the **schema** layer — and the users/roles that own and access it — actually works day to day.

---

## 2. Creating a User and an Owned Database

```sql
CREATE USER kfc_user WITH PASSWORD 'kfc_user';
CREATE DATABASE kfc_db WITH OWNER kfc_user;
```

`CREATE USER` is functionally identical to `CREATE ROLE ... WITH LOGIN` — a "user" in PostgreSQL is simply a role that's been granted the ability to log in; there's no separate underlying object type. Naming an explicit `OWNER` at database-creation time means that user automatically has full control over the new database, without needing a separate `ALTER DATABASE ... OWNER TO` step afterward.

```bash
psql -U kfc_user -d kfc_db
```
```sql
CREATE SCHEMA kfc_user;
CREATE TABLE emp(id int, sal int);
```
```
  Schema  | Name | Type  |  Owner   | Persistence | Access method |  Size   |
----------+------+-------+----------+-------------+---------------+---------+
 kfc_user | emp  | table | kfc_user | permanent   | heap          | 0 bytes |
```

The new table landed in the `kfc_user` schema **without ever explicitly qualifying it** (`kfc_user.emp`) — that's not a coincidence; Section 3 explains why.

---

## 3. A Schema Named After Its Owner — Why That Matters

PostgreSQL's default `search_path` is `"$user", public` — the special token `"$user"` means *"look first for a schema whose name exactly matches the connecting user's name."* By deliberately creating a schema **named identically to the user** (`kfc_user` created a schema also called `kfc_user`), every subsequent unqualified `CREATE TABLE` or query from that user resolves into that schema automatically, with zero explicit schema-qualification needed. This is a common, deliberate PostgreSQL convention — the closest practical equivalent to Oracle's "schema = user" mental model, achieved here by naming choice rather than being an automatic language rule.

---

## 4. Creating a Read-Only User — the Three-Step Grant

```sql
\c - postgres
CREATE USER kfc_read WITH PASSWORD 'kfc_read';

GRANT CONNECT ON DATABASE kfc_db TO kfc_read;
GRANT USAGE ON SCHEMA kfc_user TO kfc_read;
```
```sql
\c - kfc_user
GRANT SELECT ON emp TO kfc_read;
```

Three genuinely separate permission layers, each of which independently blocks access if missing:
1. **`CONNECT`** — permission to open a connection to the database at all. (`PUBLIC` is actually granted `CONNECT` on every database by default unless explicitly revoked — this step is really about being explicit/intentional, and matters most once `PUBLIC`'s default access has been locked down.)
2. **`USAGE` on the schema** — permission to even "look into" the schema's namespace. Without this, table-level grants are irrelevant — the user can't resolve the table's existence at all.
3. **`SELECT` on the specific table(s)** — the actual data-access grant.

All three are required together — missing any one of them results in a permission-denied error, even if the other two are correctly in place. This is the standard, minimum pattern for any read-only access grant.

---

## 5. GRANT ... ON ALL TABLES Is Not Retroactive

```sql
GRANT SELECT ON ALL TABLES IN SCHEMA kfc_user TO kfc_read;
```

This grants `SELECT` on every table that **already exists** in `kfc_user` at the moment the command runs — a one-time, point-in-time grant. Any table created *after* this command runs is **not** automatically covered, even though it lives in the same schema. That gap is exactly what Section 6 closes.

---

## 6. ALTER DEFAULT PRIVILEGES — Covering Future Tables

```sql
ALTER DEFAULT PRIVILEGES IN SCHEMA kfc_user GRANT SELECT ON TABLES TO kfc_read;
```

This doesn't grant anything on any table directly — it changes what happens **automatically at the moment a new table is created** in that schema going forward: any table subsequently created will have `SELECT` for `kfc_read` applied to it immediately, with no separate manual `GRANT` step required per new table.

**A real, sharp-edged caveat:** `ALTER DEFAULT PRIVILEGES` only affects objects subsequently created **by the same role that ran this command**. Since `kfc_user` ran it, only tables `kfc_user` personally creates going forward will automatically pick up this grant — a table created by some *other* user (even within the same `kfc_user` schema) would **not** automatically inherit it, unless that other user is targeted explicitly via `ALTER DEFAULT PRIVILEGES FOR ROLE <that other user> IN SCHEMA ...`.

---

## 7. Multiple Schemas and search_path

```sql
\dn
```
```
   Name   |       Owner
----------+-------------------
 kfc_user | kfc_user
 public   | pg_database_owner
```
```sql
CREATE SCHEMA kfc;
```
```
   Name   |       Owner
----------+-------------------
 kfc      | kfc_user
 kfc_user | kfc_user
 public   | pg_database_owner
```

```sql
CREATE TABLE t1 (id int, sal int);            -- lands in kfc_user, via $user matching (Section 3)
CREATE TABLE kfc.t2(id int, sal int);          -- explicitly qualified into the kfc schema
```
```sql
\dt
```
```
  Schema  | Name | Type  |  Owner
----------+------+-------+----------
 kfc_user | emp  | table | kfc_user
 kfc_user | t1   | table | kfc_user
```

**`t2` doesn't appear.** `\dt` (and unqualified queries generally) only look through schemas that are actually **in the current `search_path`** — and `kfc` was never added to it. The table genuinely exists (`kfc.t2`, fully qualified, would find it); it's simply invisible to anything relying on the default resolution path.

```sql
SET search_path = kfc, kfc_user;
\dt
```
```
  Schema  | Name | Type  |  Owner
----------+------+-------+----------
 kfc      | t2   | table | kfc_user
 kfc_user | emp  | table | kfc_user
 kfc_user | t1   | table | kfc_user
```

Once `kfc` is added to `search_path`, `t2` becomes visible — `search_path` is genuinely the list of schemas a session searches through, in order, for any unqualified object reference, not just a cosmetic display filter.

---

## 8. Same Table Name, Different Schemas — Resolution Order

```sql
CREATE TABLE kfc_user.t2(id int, sal int);   -- now t2 exists in BOTH kfc and kfc_user
```

```sql
SET search_path = kfc, kfc_user;
\dt t2
```
```
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 kfc    | t2   | table | kfc_user
```

```sql
SET search_path = kfc_user, kfc;
\dt t2
```
```
  Schema  | Name | Type  |  Owner
----------+------+-------+----------
 kfc_user | t2   | table | kfc_user
```

The core rule, demonstrated unambiguously: when the same table name exists in more than one schema on the active `search_path`, an **unqualified reference always resolves to whichever schema appears first** in `search_path` — not alphabetically, not by creation date, purely by search-path position. Reversing the order of the exact same two schema names changes which physical table `t2` refers to, with no change to the query itself. Worth understanding before relying on unqualified table names in any multi-schema application — an ambiguous name isn't rejected with an error, it's silently resolved by this ordering rule.

---

## 9. Activity — Role-Based Access Control

**The goal, stated up front:**
- Create two roles, `kfc_dba` and `kfc_dev`.
- Grant `SELECT` on all tables to `kfc_dev`.
- Grant all privileges on all tables to `kfc_dba`.
- Create four users: `kfc_dba1`, `kfc_dba2`, `kfc_dev1`, `kfc_dev2`.
- Assign the `kfc_dev` role to `kfc_dev1`/`kfc_dev2`, and `kfc_dba` to `kfc_dba1`/`kfc_dba2`.
- Audit their activity.

**Step 1 — create the two group roles, deliberately not login-capable:**
```sql
CREATE ROLE kfc_dba;
CREATE ROLE kfc_dev;
```
Created with plain `CREATE ROLE` (not `CREATE USER`) — these two exist purely as **permission containers** ("groups," in the everyday sense), not accounts anyone logs in with directly. A direct parallel to Oracle's role-based privilege grouping.

**Step 2 — grant the intended privileges to each role:**
```sql
-- kfc_dev: read-only access
GRANT CONNECT ON DATABASE kfc_db TO kfc_dev;
GRANT USAGE ON SCHEMA kfc_user TO kfc_dev;
GRANT SELECT ON ALL TABLES IN SCHEMA kfc_user TO kfc_dev;

-- kfc_dba: full access
GRANT CONNECT ON DATABASE kfc_db TO kfc_dba;
GRANT USAGE ON SCHEMA kfc_user TO kfc_dba;
GRANT ALL ON ALL TABLES IN SCHEMA kfc_user TO kfc_dba;   -- see Section 10
```

**Step 3 — create the four real login users, and assign each into the correct role:**
```sql
CREATE USER kfc_dev1;
CREATE USER kfc_dev2;
CREATE USER kfc_dba1;
CREATE USER kfc_dba2;

GRANT kfc_dev TO kfc_dev1;
GRANT kfc_dev TO kfc_dev2;
GRANT kfc_dba TO kfc_dba1;
GRANT kfc_dba TO kfc_dba2;
```

`GRANT <role> TO <user>` is PostgreSQL's **role membership/inheritance** mechanism — `kfc_dev1` and `kfc_dev2` don't get their own individual `SELECT` grants at all; they inherit every privilege `kfc_dev` holds automatically, by virtue of membership. That's the entire point of the pattern: manage privileges once, on the group role; onboarding or offboarding an individual person becomes a single `GRANT`/`REVOKE` of role membership, rather than re-deriving and re-applying a whole privilege set per person.

---

## 10. A Likely Copy-Paste Error in the Source Steps

Worth flagging directly, since it doesn't match the stated goal at the top of the activity. The `kfc_dba` grant block, as recorded, reads:

```sql
grant connect on database kfc_db to kfc_dba;
grant usage on schema kfc_user to kfc_dba;
grant all on all tables in schema kfc_user to kfc_dev;   -- <-- this line says kfc_dev
```

The stated goal was "grant all privileges on all tables to kfc_dba" — but the third line here re-targets `kfc_dev`, immediately after two lines correctly targeting `kfc_dba`. As written, this would mean **`kfc_dev` ends up with full (`ALL`) privileges** — identical to `kfc_dba` — while `kfc_dba` never actually receives its table-level grant at all (it only got `CONNECT` and `USAGE`). This reads like a copy-paste slip (reusing the `kfc_dev` target from the block above it) rather than something intentional. The corrected line should be:

```sql
GRANT ALL ON ALL TABLES IN SCHEMA kfc_user TO kfc_dba;
```

Worth checking your actual environment against this before treating the original sequence as correct — if it ran as written, `kfc_dev` currently has full privileges it shouldn't, and `kfc_dba` is currently missing the table-level grants it's supposed to have.

---

## 11. Per-User Statement Auditing

```sql
ALTER USER kfc_dba1 SET log_statement = 'all';
ALTER USER kfc_dba2 SET log_statement = 'all';
ALTER USER kfc_dev1 SET log_statement = 'all';
ALTER USER kfc_dev2 SET log_statement = 'all';
```

Ties back to the earlier logger-process material: `ALTER USER ... SET log_statement = 'all'` enables full statement logging **scoped to that specific user only** — every session that user opens gets every statement logged, while every other user (including any shared application account) is unaffected. This is the "audit the humans without flooding the log with application traffic" pattern, applied here to a full DBA/developer team rather than a single manual-access account.

```sql
ALTER USER kfc_dev1 SET search_path = kfc_user;
ALTER USER kfc_dev2 SET search_path = kfc_user;
ALTER USER kfc_dba1 SET search_path = kfc_user;
ALTER USER kfc_dba2 SET search_path = kfc_user;
```

`ALTER USER ... SET <parameter>` persists a session-level default **per user**, applied automatically every time that specific user connects — here, ensuring every one of these four accounts lands with the correct `search_path` by default, without each person needing to remember to `SET` it manually every session.

---

## 12. Decoding log_line_prefix

```ini
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
```

| Escape | Meaning |
|---|---|
| `%t` | Timestamp |
| `%p` | Process ID (the OS-level PID of the user backend process handling this session) |
| `%l` | Log line number — a simple incrementing counter of log lines *for that session* |
| `%u` | Connected user name |
| `%d` | Connected database name |
| `%a` | Application name (as reported by the connecting client — `psql`, an app's connection string, etc.) |
| `%h` | Client host/address the connection originated from |

**On `[%l-1]` specifically:** the `-1` is **literal text**, not a special escape sequence — the prefix string always appends the literal characters `-1` immediately after whatever number `%l` substitutes in. Just a formatting convention some teams use, not PostgreSQL-defined behavior.

**Resulting real log line, once `log_statement = 'all'` picked up a statement:**
```
2026-05-01 08:17:38 IST [7137]: [6-1] user=kfc_user,db=kfc_db,app=psql,client=[local] LOG:  statement: delete from emp;
```

Reading it field by field: timestamp `2026-05-01 08:17:38 IST`; process ID `7137`; this session's 6th logged line (`[6-1]`); connected as `kfc_user`, to database `kfc_db`, via the `psql` client, from `[local]` (a local Unix-socket connection, not a remote TCP client); and finally the captured statement itself — `DELETE FROM emp;`. This is exactly what makes per-user auditing useful in practice — every logged statement is immediately attributable to a specific user, database, and originating client, not just a bare SQL string.
