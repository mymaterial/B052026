# psql Deep Dive — Session Notes

This session is a focused, hands-on deep dive into `psql` itself — connection strings and defaults, meta-commands as shortcuts to catalog views, password handling (`.pgpass`), the `.psqlrc` startup file, and the command-line flags used for scripting (`-c`, `-f`, `-X`, `-a`). The session's original agenda also included physical files (data files), WAL files, the control file, and startup/shutdown modes — none of that was reached; the full session stayed on `psql`. Those topics will need to be picked up in a future session.

---
## Table of Contents

- [1. psql Connection String and Defaults](#1-psql-connection-string-and-defaults)
- [2. Meta-Commands Are Shortcuts to Catalog Views](#2-meta-commands-are-shortcuts-to-catalog-views)
- [3. log_statement for Tracing Executed SQL](#3-log_statement-for-tracing-executed-sql)
- [4. Password Handling — Inline, Environment Variable, and .pgpass](#4-password-handling--inline-environment-variable-and-pgpass)
- [5. .psqlrc — Custom Startup File](#5-psqlrc--custom-startup-file)
- [6. Suppressing .psqlrc at Login (-X)](#6-suppressing-psqlrc-at-login--x)
- [7. Scripting with -c and -f](#7-scripting-with--c-and--f)
- [8. Echoing Input Alongside Output (-a)](#8-echoing-input-alongside-output--a)
- [9. Questions Discussed in This Session](#9-questions-discussed-in-this-session)

---

## 1. psql Connection String and Defaults

Every `psql` tool ships with a `--help` option listing its available flags:

```bash
psql --help
```

**If no connection details are passed, `psql` falls back to defaults:**

| Detail | Default |
|---|---|
| Username | `postgres` |
| Database | `postgres` |
| Host | local socket connection |
| Port | `5432` |
| Password | governed by `pg_hba.conf` (`trust` on many lab setups — no password prompt at all) |

```bash
psql
# connects as user "postgres" to database "postgres" on localhost:5432
```

To override any of these, pass the corresponding flag explicitly:

```bash
psql -U someuser -d somedb -h somehost -p someport
```

**Prompt symbol tells you the connection's privilege level:**
- `#` — connected as a superuser.
- `>` — connected as a regular (non-superuser) user.

---

## 2. Meta-Commands Are Shortcuts to Catalog Views

Every `psql` backslash meta-command (`\l`, `\du`, `\di`, etc.) is a convenience shortcut that, behind the scenes, queries the underlying system catalog view(s) directly.

| Meta-command | What it shows | Underlying catalog |
|---|---|---|
| `\l` | List of databases | `pg_database` |
| `\du` | List of roles/users | `pg_roles` |
| `\di` | List of indexes | `pg_index`, joined with `pg_class` (index shortcuts are almost always the relevant catalog view joined with `pg_class` for the human-readable name) |
| `\?` | List of all meta-commands | — |
| `\h` | SQL command syntax help | — (unreliable for anything beyond basic syntax; official documentation is the more practical reference for real work) |

Anything a meta-command shows can also be queried directly from its underlying catalog view — the meta-command is purely a typing shortcut, not a separate mechanism:

```sql
SELECT * FROM pg_database;    -- same information as \l
```

---

## 3. log_statement for Tracing Executed SQL

```ini
# postgresql.conf
log_statement = all
```

With this enabled, every SQL statement executed against the server is written to the log. This setting takes effect on a config reload — it does **not** require a full server restart.

```bash
pg_ctl reload -D /path/to/data
# or: SELECT pg_reload_conf();
```

---

## 4. Password Handling — Inline, Environment Variable, and .pgpass

**Interactive prompt** (default): `psql` prompts for a password if the matching `pg_hba.conf` rule requires one (e.g. `md5`).

**Environment variable:**

```bash
export PGPASSWORD=your_password
psql
```

**`.pgpass` file** — the standard way to avoid hardcoding a password directly into a script:

```
# ~/.pgpass — format: hostname:port:database:username:password
localhost:5432:*:postgres:your_password
```

```bash
chmod 600 ~/.pgpass   # required — psql will ignore the file with wrong permissions
psql
# connects without prompting for a password
```

Both `PGPASSWORD` and `.pgpass` exist specifically for **automated/scripted connections** (cron jobs, batch scripts) where interactively typing a password isn't possible and hardcoding a plaintext password directly in a script is undesirable.

---

## 5. .psqlrc — Custom Startup File

`psql` reads a startup file (`~/.psqlrc` by default) every time it starts, which can customize the prompt, define shortcuts, and pre-run queries into psql variables.

```sql
-- ~/.psqlrc
\set PROMPT1 '%n@%/%R%# '
\echo 'Welcome — quick reference:'
\echo '  version  -> show PostgreSQL version'
\echo '  extension -> list available extensions'
\echo '  locks    -> show current lock information'

SELECT version() \gset
SELECT string_agg(name, ', ') FROM pg_available_extensions \gset extension
```

Custom shortcuts for frequently used diagnostic queries (logs, long-running queries, replication lag, current locks, and similar) are commonly kept this way — sometimes referred to informally as "login scripts," similar in spirit to a custom `.sqlrc` some people set up for other database clients.

```sql
-- example custom shortcut defined in .psqlrc
\set locks 'SELECT * FROM pg_locks;'
```

```bash
psql
:locks    -- runs the locks query
```

---

## 6. Suppressing .psqlrc at Login (-X)

If `.psqlrc` prints information you don't want on every login (echoed messages, custom banners), suppress it for a single session with `-X`:

```bash
psql -X
# connects without processing ~/.psqlrc at all
```

---

## 7. Scripting with -c and -f

**`-c` — run a single command and exit:**

```bash
psql -c "SELECT * FROM employee;"
```

Multiple `-c` flags can be chained to run several statements in sequence, each as its own implicit transaction:

```bash
psql -c "INSERT INTO employee VALUES (2, 300);" -c "SELECT * FROM employee;"
```

**`-f` — run all statements from a SQL file:**

```sql
-- select.sql
SELECT * FROM employee;
SELECT now();
```

```bash
psql -f select.sql
```

---

## 8. Echoing Input Alongside Output (-a)

By default, `psql -f` prints only the query **results**, not the queries themselves — which makes output harder to follow when a file has several statements. Add `-a` to echo each input statement immediately before its output:

```bash
psql -a -f select.sql
```

```
SELECT * FROM employee;
 id | salary
----+--------
  1 |    100
  2 |    300

SELECT now();
              now
-------------------------------
 2026-05-31 09:30:00.123456+05:30
```

---

## 9. Questions Discussed in This Session

**Q1. What are the two flags used to run queries non-interactively, and what's the difference between them?**

`-c` runs a single inline command (or several, if chained with multiple `-c` flags) and exits. `-f` runs every statement contained in a given SQL file and exits. `-c` is for one-off or scripted individual commands; `-f` is for a saved, reusable set of statements.

---

**Q2. Is there a flag to stop psql from processing `.psqlrc` at login?**

Yes — `psql -X`. This skips reading `~/.psqlrc` entirely for that session, useful when the startup file's custom output (banners, echoed variables) isn't wanted for a particular connection.

---

**Q3. When running `psql -f` on a file with multiple statements, the output only shows results, not the queries that produced them — is there a way to see both?**

Yes — add `-a` (echo all input). This prints each statement from the file immediately above its corresponding result, making multi-statement file output much easier to read.

---

**Q4. Does `log_statement = all` require a server restart to take effect?**

No — it's a "reload"-context parameter, so `pg_ctl reload` (or `SELECT pg_reload_conf();`) is sufficient. This distinction (reload vs. full restart) is worth checking per-parameter, since some `postgresql.conf` settings do require a full restart (e.g. `listen_addresses`, covered in a different session) while others, like this one, take effect immediately on reload.
