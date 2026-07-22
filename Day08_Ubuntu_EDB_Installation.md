# PostgreSQL Installation — Study Notes & Q&A
### Session 3 | PostgreSQL Production Labs Course

---

## 1. Installation Methods — Overview

PostgreSQL can be installed using three methods. In real-world production systems, the **Package Method (RPM/DEB)** is used almost exclusively.

| Method | Used in Production? | Notes |
|---|---|---|
| Package Method (RPM/DEB) | ✅ Yes — 90% of cases | Standard. Uses OS package manager (dnf/apt). |
| Root Approach (systemctl) | ✅ Yes — with custom setup | Registers PostgreSQL as an OS service. |
| Source / Manual Compile | ❌ Rarely | Only for custom builds or testing. |

---

## 2. Installation on CentOS / RHEL

### 2.1 The 5-Step Package Method

These 5 commands are the standard approach. Memorise them.

**Step 1** — Add the PGDG repository (PostgreSQL official repo)
```bash
dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-<ver>-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

**Step 2** — Disable the built-in AppStream module to avoid version conflicts
```bash
dnf -qy module disable postgresql
```

**Step 3** — Install the PostgreSQL server software
```bash
dnf install -y postgresql18-server
```

**Step 4** — Initialize the database cluster (creates data files)
```bash
/usr/pgsql-18/bin/postgresql-18-setup initdb
```

**Step 5** — Enable and start the service
```bash
systemctl enable postgresql-18 --now
```

> ⚠️ After Step 5, PostgreSQL is registered as an OS service. Always use `systemctl` — never `pg_ctl` — to start/stop the cluster.

---

### 2.2 PGDG Repository vs AppStream

Two sources exist for PostgreSQL packages:

- **AppStream** (CentOS built-in) — ships an older version (e.g. PG 13). Updated only when the OS updates.
- **PGDG Repository** (PostgreSQL official) — always has the latest version. This is what you want.

When you see `pgdg` in the install output, it confirms the package is coming from the official PostgreSQL repository, not AppStream.

---

### 2.3 Choosing a Custom Data Directory

Default data directory: `/var/lib/pgsql/18/data`

To use a custom location (recommended in production):

1. Open the systemd service file
```bash
vi /usr/lib/systemd/system/postgresql-18.service
```

2. Find the `PGDATA` line and change it
```
Environment=PGDATA=/u01/pgsql/18/data
```

3. Reload the daemon
```bash
systemctl daemon-reload
```

4. Initialize the cluster at the new location
```bash
/usr/pgsql-18/bin/initdb -D /u01/pgsql/18/data
```

5. Start the service
```bash
systemctl start postgresql-18
```

---

### 2.4 Running Multiple Clusters on CentOS

Each cluster needs its own **service file** and its own **port**.

1. Copy the existing service file with a new name
```bash
cp /usr/lib/systemd/system/postgresql-18.service \
   /usr/lib/systemd/system/postgresql-18-secondary.service
```

2. Edit the copy — change `PGDATA` to a new directory
```
Environment=PGDATA=/u01/pgsql/data1
```

3. Reload the daemon
```bash
systemctl daemon-reload
```

4. Initialize the new cluster
```bash
/usr/pgsql-18/bin/initdb -D /u01/pgsql/data1
```

5. Change the port in the new cluster's `postgresql.conf`
```
port = 5433   # each cluster must use a unique port
```

6. Start the new service
```bash
systemctl start postgresql-18-secondary
```

> ✅ Repeat this pattern for as many clusters as needed.

---

### 2.5 systemctl vs pg_ctl

| Tool | When to Use | Oracle Equivalent |
|---|---|---|
| `systemctl` | When PostgreSQL was installed via root/package method | SRVCTL |
| `pg_ctl` | When managing clusters not registered as OS services | SQL*Plus startup/shutdown |

> ⚠️ Never mix `systemctl` and `pg_ctl` on the same cluster. If you start with `systemctl` and stop with `pg_ctl`, the OS service manager still thinks the cluster is running.

---

### 2.6 Granting Non-root Users systemctl Access

In production, the `postgres` OS user is given limited sudo rights:

```bash
# Add to /etc/sudoers:
postgres ALL=(ALL) NOPASSWD: /bin/systemctl start postgresql-18, /bin/systemctl stop postgresql-18
```

---

## 3. Installation on Ubuntu

### 3.1 Ubuntu-Specific Cluster Tools

Ubuntu ships extra cluster-management utilities not found in CentOS. Install them first:

```bash
apt install postgresql-common
```

| Command | Purpose |
|---|---|
| `pg_createcluster <ver> <name>` | Create a new PostgreSQL cluster |
| `pg_lsclusters` | List all clusters and their status/ports |
| `pg_dropcluster <ver> <name>` | Remove a cluster |
| `pg_ctlcluster <ver> <name> start/stop` | Start/stop a specific cluster |

---

### 3.2 Ubuntu Installation Steps (3 Steps Only)

**Step 1** — Add PGDG repository
```bash
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
```

**Step 2** — Install the software
```bash
sudo apt install -y postgresql-18
```

**Step 3** — Verify (cluster is already initialized and running)
```bash
pg_lsclusters
```

> Unlike CentOS, Ubuntu automatically runs `initdb` and starts the cluster after package installation.

---

### 3.3 Multiple Clusters on Ubuntu

```bash
pg_createcluster 18 main2
pg_createcluster 18 main3
pg_lsclusters   # shows all clusters with auto-assigned ports
```

Ports are assigned automatically (5432, 5433, 5434…). No manual port editing required.

---

### 3.4 CentOS vs Ubuntu — Comparison

| Feature | CentOS / RHEL | Ubuntu |
|---|---|---|
| Package manager | dnf / yum | apt |
| initdb | Manual (step 4 of 5) | Automatic on install |
| Service registration | Manual (`systemctl enable`) | Automatic |
| Custom data directory | Edit service file → daemon-reload | Edit cluster config file |
| Multiple clusters | Copy service file + change port | `pg_createcluster` (one command) |
| Port assignment | Manual (edit postgresql.conf) | Automatic, incremental |
| Recommended for | Single cluster, production standard | Multi-cluster, dev/learning |

---

## 4. EDB Advanced Server — Oracle-Compatible PostgreSQL

### 4.1 What Is EDB?

EDB (EnterpriseDB) is a commercial distribution of PostgreSQL with two products:

- **Postgres Plus Advanced Server** — Oracle-compatible flavor (most commonly used)
- **Postgres Plus Extended Server** — performance-focused flavor

EDB's positioning: just as AWS claims "cloud-optimized PostgreSQL", EDB claims "Oracle-optimized PostgreSQL".

---

### 4.2 Why EDB for Oracle Migrations?

EDB reduces Oracle → PostgreSQL migration effort by ~60–70% by natively supporting:

- Oracle PL/SQL anonymous blocks (`BEGIN … END` syntax)
- `NVL()` function (community PG uses `COALESCE`)
- `DBMS_OUTPUT.PUT_LINE` (community PG uses `RAISE NOTICE`)
- Oracle data types and common built-in functions
- `db_dialect = redwood` setting for Oracle-compatible mode

---

### 4.3 Code Comparison

**Oracle (source):**
```sql
DECLARE
  v_avg  NUMBER;
  v_avg2 NUMBER;
BEGIN
  SELECT AVG(salary) INTO v_avg FROM employee;
  SELECT AVG(NVL(salary, 0)) INTO v_avg2 FROM employee;
  DBMS_OUTPUT.PUT_LINE('Avg: ' || v_avg);
  DBMS_OUTPUT.PUT_LINE('Avg with NVL: ' || v_avg2);
END;
```

**EDB Advanced Server** — ✅ The Oracle code above runs without modification.

**Community PostgreSQL** — manual rewrites required:
```sql
DO $$
DECLARE
  v_avg  NUMERIC;
  v_avg2 NUMERIC;
BEGIN
  SELECT AVG(salary) INTO v_avg FROM employee;
  SELECT AVG(COALESCE(salary, 0)) INTO v_avg2 FROM employee;
  RAISE NOTICE 'Avg: %', v_avg;
  RAISE NOTICE 'Avg with COALESCE: %', v_avg2;
END $$;
```

**Syntax translation table:**

| Oracle | Community PostgreSQL |
|---|---|
| `NVL(col, default)` | `COALESCE(col, default)` |
| `DBMS_OUTPUT.PUT_LINE('msg')` | `RAISE NOTICE 'msg';` |
| `NUMBER` | `NUMERIC` |
| `BEGIN … END;` (no $$) | `DO $$ BEGIN … END $$;` |
| `DECLARE` before `BEGIN` | `DECLARE` inside `DO $$` block |

---

## 5. Q&A — Test Your Understanding

---

**Q1. What is the default data directory when PostgreSQL is installed using the package method on CentOS?**

`/var/lib/pgsql/18/data`

This location is defined in the systemd service file and used by `initdb` when no custom location is specified.

---

**Q2. What is the difference between AppStream and PGDG repository?**

AppStream is the CentOS built-in repo — it ships an older PG version and only updates when the OS does. PGDG is the official PostgreSQL repo and always has the latest version. When you see `pgdg` in install output, it confirms the package came from the official source.

---

**Q3. You installed PostgreSQL using the package method and started it with `systemctl`. Your colleague then runs `pg_ctl stop`. What is the problem?**

The OS service manager (systemd) still believes the cluster is running because it did not receive the stop signal through `systemctl`. This causes a state mismatch — the cluster is actually down but `systemctl status` will show it as active. Rule: never mix `systemctl` and `pg_ctl` on the same cluster.

---

**Q4. How do you create a second PostgreSQL cluster on CentOS?**

1. Copy the service file with a new name
2. Edit the copy — change `PGDATA` to a new directory
3. `systemctl daemon-reload`
4. `initdb` on the new directory
5. Edit `postgresql.conf` in the new directory — change the port (e.g. 5433)
6. `systemctl start <new-service-name>`

---

**Q5. Why is multi-cluster setup easier on Ubuntu than CentOS?**

Ubuntu's `pg_createcluster` utility handles everything in one command: service file creation, `initdb`, and automatic port assignment. On CentOS, all of this must be done manually across multiple steps.

---

**Q6. What does `systemctl daemon-reload` do, and when must you run it?**

It tells systemd to re-read all service files from disk. You must run it whenever you create or edit a service file. Without it, systemd keeps using the cached (old) version of the file and ignores your changes.

---

**Q7. A developer has Oracle PL/SQL code using `NVL()` and `DBMS_OUTPUT.PUT_LINE()`. What are their migration options?**

- **Community PostgreSQL** — rewrite manually: replace `NVL()` with `COALESCE()`, `DBMS_OUTPUT.PUT_LINE()` with `RAISE NOTICE`, add `$$` delimiters, change `NUMBER` to `NUMERIC`.
- **EDB Advanced Server** — the Oracle code runs as-is in most cases, reducing migration effort by ~60–70%.

---

**Q8. What is `db_dialect = redwood` in EDB?**

A configuration setting in EDB Advanced Server that activates Oracle-compatibility mode. When set to `redwood`, EDB interprets SQL and PL/SQL with Oracle semantics. This is the default mode in EDB Advanced Server.

---

**Q9. After installing PostgreSQL on Ubuntu, do you need to run `initdb` manually?**

No. Ubuntu automatically runs `initdb` and starts the cluster as part of the package installation. Verify with `pg_lsclusters`.

---

**Q10. Translate this Oracle snippet to community PostgreSQL:**
```sql
DBMS_OUTPUT.PUT_LINE('Value: ' || my_var);
```

```sql
RAISE NOTICE 'Value: %', my_var;
```

Key difference: `RAISE NOTICE` uses `%` as a positional placeholder rather than string concatenation (`||`).

---

## 6. Command Quick Reference

| Task | CentOS | Ubuntu |
|---|---|---|
| Add repo | `dnf install pgdg-redhat-repo-latest...` | `apt.postgresql.org.sh` script |
| Install PG | `dnf install -y postgresql18-server` | `apt install -y postgresql-18` |
| Initialize cluster | `postgresql-18-setup initdb` | Automatic |
| Start cluster | `systemctl start postgresql-18` | `systemctl start postgresql@18-main` |
| Stop cluster | `systemctl stop postgresql-18` | `systemctl stop postgresql@18-main` |
| Status | `systemctl status postgresql-18` | `pg_lsclusters` |
| Add new cluster | Copy service file + initdb + port change | `pg_createcluster 18 <name>` |
| List clusters | `ps -ef \| grep postgres` | `pg_lsclusters` |
| Connect | `psql -U postgres` | `psql -U postgres` |
| Connect to EDB | `psql -p 5444 -d edb` | N/A |

---

*End of Session 3 Study Material — PostgreSQL Production Labs*
