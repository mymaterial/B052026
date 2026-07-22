# Connection Pooling, Roles & Auditing, Stored Procedure Privileges — Session Notes

This session opens with a real-world troubleshooting discussion around a Kubernetes-hosted application's connection pooling issue, then returns to the user-management curriculum: the practical difference between `CREATE ROLE` and `CREATE USER`, building a role-based access control (RBAC) structure for DBA/developer teams, auditing manual activity via `log_statement`, telling reload-safe parameters from restart-required ones, and a hands-on test of how stored-procedure `EXECUTE` privilege interacts with privileges on the tables a procedure touches.

---
## Table of Contents

- [1. Real-World Troubleshooting — Connection Pooling on GKE](#1-real-world-troubleshooting--connection-pooling-on-gke)
- [2. Application-Level Connection Pooling Demo (HikariCP)](#2-application-level-connection-pooling-demo-hikaricp)
- [3. CREATE ROLE vs CREATE USER](#3-create-role-vs-create-user)
- [4. Building a Role-Based Access Control Structure](#4-building-a-role-based-access-control-structure)
- [5. Auditing Manual Activity](#5-auditing-manual-activity)
- [6. Reload vs Restart — Reading the context Column](#6-reload-vs-restart--reading-the-context-column)
- [7. Stored Procedures and EXECUTE Privilege](#7-stored-procedures-and-execute-privilege)
- [8. The ALTER DEFAULT PRIVILEGES Gotcha, Revisited](#8-the-alter-default-privileges-gotcha-revisited)
- [9. Questions Discussed in This Session](#9-questions-discussed-in-this-session)

---

## 1. Real-World Troubleshooting — Connection Pooling on GKE

A student (Srini) brought a live production issue for discussion: an application running on Google Kubernetes Engine (GKE), connecting to a vanilla PostgreSQL 18 instance, was experiencing intermittent connection timeouts that looked like a firewall/networking problem rather than a database configuration problem.

**Points established during the discussion:**
- A connection **timeout** (as opposed to an immediate "connection refused") almost always points to a network path problem — the request is being sent but never reaching (or never getting a response from) the target, rather than being actively rejected. This is a different signature from a `pg_hba.conf` rejection or an authentication failure, both of which return immediately with a clear error.
- Non-default ports (the environment in question used `5435` instead of `5432`) need to be explicitly allowed at every layer in the path — firewall rules, security groups, and any network policies — not just in PostgreSQL's own configuration. A working `listen_addresses`/`pg_hba.conf` setup is necessary but not sufficient if an intermediate firewall is silently dropping traffic on the custom port.
- When multiple teams/layers are involved (Kubernetes networking, firewall/network team, application team, DBA), the most useful first diagnostic step is getting the actual client-side error message rather than guessing — "connection timeout," "connection refused," and an authentication error each point to a completely different layer of the stack, and mean the fix belongs to a different team.
- The general recommendation: get the exact error message first, work outward from there (application → connection pooler → network/firewall → database), and loop in the responsible team early rather than guessing across layers.

---

## 2. Application-Level Connection Pooling Demo (HikariCP)

To make connection pool behavior concrete, a live demo was run using **HikariCP** (the connection pool built into Spring Boot applications) against `pg_stat_activity`:

```sql
SELECT pid, query FROM pg_stat_activity;
```

**Configured pool behavior in the demo:**
- Maximum pool size: 10 active connections at a time.
- A 20-connection burst was simulated (representing 20 near-simultaneous client requests).
- HikariCP served the first 10 immediately; the remaining 10 queued and waited.
- Once the first batch completed, the pool served the second batch of 10 from the queue.
- After an idle period, the pool released connections back down toward a configured minimum (2, in the demo), rather than holding all 10 open indefinitely.

**Key point:** the connection pool's port/connection-string settings are completely independent of the pool's *sizing* behavior — pool size, minimum idle connections, and connection timeout are pool-level configuration (in this case, in the Spring Boot application's `application.yaml`, or hardcoded in application startup code), not something PostgreSQL itself is aware of or controls. If a team needs the pool size changed and doesn't have code access, the request has to go through whichever mechanism deploys the application (e.g. asking the team to modify their `application.yaml`, or the CI/CD script that launches it) — this isn't a PostgreSQL-side change.

---

## 3. CREATE ROLE vs CREATE USER

```sql
CREATE USER test_a;   -- output: CREATE ROLE
CREATE ROLE test_b;   -- output: CREATE ROLE
```

Both commands produce identical output (`CREATE ROLE`) because, internally, PostgreSQL has only one concept — a **role**. `CREATE USER` and `CREATE ROLE` are the same underlying command with different **defaults**:

| | `CREATE USER` | `CREATE ROLE` |
|---|---|---|
| `LOGIN` attribute | Included by default (can connect/authenticate) | **Not** included by default (cannot log in unless `WITH LOGIN` is added explicitly) |

**A role is nothing more than a named set of privileges.** Whether that role can also be used to log in is just one attribute (`LOGIN`) on top of that same underlying object. The keywords `USER` and `ROLE` are interchangeable in PostgreSQL's SQL syntax beyond that one default — there's no separate "user object" and "role object" under the hood.

---

## 4. Building a Role-Based Access Control Structure

**Scenario:** a team of 2 DBAs and 2 developers, each with individual logins, where DBAs get full privileges and developers get read-only access — set up via roles rather than granting privileges to each person individually.

```sql
-- as postgres: create the two roles (non-login groups of privileges)
CREATE ROLE kfc_dba;
CREATE ROLE kfc_dev;

-- as the schema owner: grant privileges to the roles, not to individual people
GRANT SELECT ON ALL TABLES IN SCHEMA kfc_user TO kfc_dev;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA kfc_user TO kfc_dba;

-- as postgres: create the individual login users
CREATE USER kfc_dba1 WITH PASSWORD '...';
CREATE USER kfc_dba2 WITH PASSWORD '...';
CREATE USER kfc_developer1 WITH PASSWORD '...';
CREATE USER kfc_developer2 WITH PASSWORD '...';

-- assign each individual user to the appropriate role — inherits all of that role's privileges in one command
GRANT kfc_dba TO kfc_dba1, kfc_dba2;
GRANT kfc_dev TO kfc_developer1, kfc_developer2;
```

Once a user is granted a role, they inherit every privilege that role has — present and future grants alike — without needing individual `GRANT` statements per person. This is the practical reason to model access as roles-per-function rather than privileges-per-person: adding a fifth developer later is one `GRANT kfc_dev TO kfc_developer5;`, not re-running every individual table grant again.

**Real-world context given:** individual per-person logins are the exception, not the norm, in most projects — most teams share a common application/schema-owner login rather than issuing each DBA/developer their own credentials. Some projects do require individual logins (often for audit/compliance reasons), which is the scenario this walkthrough models.

---

## 5. Auditing Manual Activity

To track exactly who ran what, per user:

```sql
ALTER USER kfc_developer1 SET log_statement TO 'all';
ALTER USER kfc_developer2 SET log_statement TO 'all';
ALTER USER kfc_dba1 SET log_statement TO 'all';
ALTER USER kfc_dba2 SET log_statement TO 'all';
```

This makes `log_statement = all` apply specifically to these users' sessions — permanently, at the role level — rather than setting it cluster-wide.

To actually identify *which* user ran a given logged statement, the log needs the username and database name in its output — controlled by `log_line_prefix`:

```ini
# postgresql.conf
log_line_prefix = '%t [%p]: [%u@%d] '
```

**Setting each audited user's `search_path` permanently**, since these individual-login users work against the shared application schema rather than creating their own objects:

```sql
ALTER USER kfc_developer1 SET search_path = kfc_user;
ALTER USER kfc_developer2 SET search_path = kfc_user;
ALTER USER kfc_dba1 SET search_path = kfc_user;
ALTER USER kfc_dba2 SET search_path = kfc_user;
```

**Verifying the setup:**

```sql
\c kfc_db -U kfc_developer1
SELECT * FROM employee;        -- works (SELECT privilege via kfc_dev role)
DELETE FROM employee;          -- ERROR: permission denied (no DELETE privilege)
INSERT INTO employee VALUES (1, 500);  -- ERROR: permission denied
```

```sql
\c kfc_db -U kfc_dba1
DELETE FROM employee WHERE id = 1;     -- succeeds (full privileges via kfc_dba role)
```

Since `log_statement = all` is enabled for `kfc_dba1`, the alert log captures exactly this statement, with a timestamp and the username attached — giving a clear audit trail of who deleted the record and when.

---

## 6. Reload vs Restart — Reading the context Column

Rather than memorizing which parameters need a restart, check the `context` column in `pg_settings` directly:

```sql
SELECT name, context FROM pg_settings
WHERE name IN ('port', 'log_statement', 'log_connections');
```

| `context` value | What it means |
|---|---|
| `postmaster` | Requires a full server **restart** to take effect |
| `sighup` | Takes effect on a config **reload** (`pg_ctl reload` / `SELECT pg_reload_conf();`) — no restart needed |
| `superuser`, `user`, `backend`, `internal`, and others | Various narrower scopes (some are session-settable via `SET`, some need superuser, etc.) — none of these require a full restart |

**Worked example from the demo:** `port` has `context = postmaster` (restart required); `log_statement` and `log_connections` do not — a reload is sufficient for both.

**A second, faster way to check** without querying `pg_settings`: open `postgresql.conf` directly and look at the inline comment next to a parameter — PostgreSQL's default config file annotates restart-required parameters with a comment like `# (change requires restart)` right next to the setting.

---

## 7. Stored Procedures and EXECUTE Privilege

```sql
CREATE PROCEDURE add_employee(p_id int, p_salary int)
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO employee (id, salary) VALUES (p_id, p_salary);
END;
$$;

CALL add_employee(1, 100);
```

**Calling this procedure as `kfc_developer2` (who only has `SELECT` on `employee`, no `EXECUTE` on the procedure) fails immediately with a permission error** — invoking any function or procedure always requires `EXECUTE` privilege on that specific procedure, granted separately from any privileges on the tables it touches:

```sql
-- as the schema owner
GRANT EXECUTE ON PROCEDURE add_employee TO kfc_developer2;
```

**Granting `EXECUTE` alone is still not enough**, because PostgreSQL procedures run with the *caller's* privileges by default (`SECURITY INVOKER` semantics, unless the procedure is explicitly declared `SECURITY DEFINER`). Since the procedure body does an `INSERT INTO employee`, the calling user also needs `INSERT` privilege on `employee` directly — `EXECUTE` on the procedure and `INSERT` on the underlying table are two separate, both-required grants:

```sql
GRANT INSERT ON employee TO kfc_developer2;
```

Only with **both** grants in place does `CALL add_employee(2, 300);` succeed as `kfc_developer2`. Neither grant substitutes for the other — a user with `INSERT` on the table but no `EXECUTE` on the procedure still cannot call it, and a user with `EXECUTE` on the procedure but no `INSERT` on the table it modifies still cannot successfully run it.

---

## 8. The ALTER DEFAULT PRIVILEGES Gotcha, Revisited

A concrete demonstration of why `ALTER DEFAULT PRIVILEGES` matters (introduced in an earlier session): `GRANT SELECT ON ALL TABLES IN SCHEMA ... TO ...` only covers tables that **already existed** at the time the grant ran.

```sql
-- as kfc_user (schema owner)
CREATE TABLE department (id int, salary int);
```

Because the earlier `GRANT SELECT ON ALL TABLES ...` command was run *before* this table existed, `kfc_developer2` has **no** access to `department` yet — despite already having `SELECT` on every other table in the schema. It has to be granted again explicitly (or `ALTER DEFAULT PRIVILEGES` needs to have been set up in advance to cover future tables automatically, as covered previously):

```sql
GRANT SELECT ON ALL TABLES IN SCHEMA kfc_user TO kfc_dev;   -- re-run, now covers department too
```

---

## 9. Questions Discussed in This Session

**Q1. If we enable auditing (`log_statement = all`) broadly, won't that generate an overwhelming amount of log data?**

Yes, if applied indiscriminately — which is why the guidance is to enable it specifically for **manual/individual-login users** (DBAs, developers who log in directly to run ad hoc work), not for the application's own service account. Application traffic typically runs at a much higher volume than manual DBA activity, so logging every application-generated statement would flood the log with noise. If a DBA occasionally has to log in *as* the application's service account (e.g. to run a deployment script without individual credentials), that's specifically when logging gets harder to keep clean — because now `log_statement` on that shared account would capture both routine application traffic and the manual activity you actually wanted to audit. If audit log volume becomes a genuine problem, narrowing `log_statement` from `all` to `mod` (statements that modify data) reduces volume while still capturing the activity of real concern.

---

**Q2. How do you tell whether a parameter change needs a restart or just a reload?**

Query the `context` column in `pg_settings` for that parameter: `context = 'postmaster'` means a restart is required; any other context value means a reload (or narrower scope, like a session-level `SET`) is sufficient. Alternatively, `postgresql.conf` itself typically annotates restart-required parameters with an inline comment.

---

**Q3. Does having `INSERT` privilege on a table make `EXECUTE` privilege on a procedure that inserts into it unnecessary, or vice versa?**

No — neither substitutes for the other. `EXECUTE` privilege on the specific procedure is always required just to invoke it at all. Separately, because procedures run with the calling user's own privileges by default (not the procedure owner's), the caller also needs whatever table-level privileges (`INSERT`, `UPDATE`, `SELECT`, etc.) the procedure's body actually requires. Both grants are independently necessary; missing either one causes a permission-denied error.

---

**Q4. If a user is granted `EXECUTE` on a procedure, can they use that access to insert data into a *different*, newly created table they don't otherwise have access to?**

No. `EXECUTE` privilege only covers invoking that specific procedure and whatever operations its body performs on the specific objects it references (subject to the caller also holding the necessary privileges on those objects, per Q3) — it grants no general ability to create tables or access arbitrary tables outside what that procedure's code touches.
