# Standard Operating Procedure
## PostgreSQL 14 — Installation via RPM Packages

> **Version:** 1.0 | **Applies to:** RHEL 9 / Rocky Linux 9 / AlmaLinux 9 (x86_64) | **Date:** May 2026  
> **Classification:** Internal Use Only

---

## 1. Overview

This SOP describes installing PostgreSQL 14.13 using offline RPM packages downloaded from the official PostgreSQL Global Development Group (PGDG) YUM repository. This method is suitable for air-gapped servers or environments where internet access is restricted at install time.

---

## 2. Scope & Applicability

- **Target OS:** RHEL 9, Rocky Linux 9, AlmaLinux 9 (x86_64)
- **PostgreSQL version:** 14.13
- **RPM source:** `download.postgresql.org` (PGDG official)
- **Install prefix:** `/usr/pgsql-14/` (fixed by RPM)
- **Data directory:** `/u01/pgsql/14`
- **OS user:** `postgres` (created automatically by the RPM)

---

## 3. RPM Package Overview

Four RPMs are required. They must be installed in dependency order:

| Install Order | Package | Description |
|:---:|---------|-------------|
| 1 | `postgresql14-libs` | Shared client libraries (libpq) — required by all other packages |
| 2 | `postgresql14` | Client tools: psql, pg_dump, pg_restore |
| 3 | `postgresql14-server` | Server binaries: postgres, initdb, pg_ctl |
| 4 | `postgresql14-contrib` | Optional extensions: pg_stat_statements, pgcrypto, etc. |

> **NOTE:** `postgresql14-libs` must be installed first. The server and contrib packages depend on it. Installing in wrong order will result in dependency errors.

---

## 4. Download RPM Packages

Download all four packages from the PGDG YUM mirror. Run the following as `root` from your staging directory:

```bash
wget https://download.postgresql.org/pub/repos/yum/14/redhat/rhel-9-x86_64/postgresql14-libs-14.13-1PGDG.rhel9.x86_64.rpm
wget https://download.postgresql.org/pub/repos/yum/14/redhat/rhel-9-x86_64/postgresql14-14.13-2PGDG.rhel9.x86_64.rpm
wget https://download.postgresql.org/pub/repos/yum/14/redhat/rhel-9-x86_64/postgresql14-server-14.13-2PGDG.rhel9.x86_64.rpm
wget https://download.postgresql.org/pub/repos/yum/14/redhat/rhel-9-x86_64/postgresql14-contrib-14.13-2PGDG.rhel9.x86_64.rpm
```

Verify all four files are present:

```bash
ls -lh postgresql14*.rpm
```

---

## 5. Install RPM Packages

### Option A — Install in Dependency Order (Recommended)

Installing individually gives you clear visibility into each step and any errors:

```bash
rpm -ivh postgresql14-libs-14.13-1PGDG.rhel9.x86_64.rpm
rpm -ivh postgresql14-14.13-2PGDG.rhel9.x86_64.rpm
rpm -ivh postgresql14-server-14.13-2PGDG.rhel9.x86_64.rpm
rpm -ivh postgresql14-contrib-14.13-2PGDG.rhel9.x86_64.rpm
```

### Option B — Install All at Once

RPM resolves the dependency order automatically when all files are passed together:

```bash
rpm -ivh *.rpm
```

> **NOTE:** Option B works only if all RPMs are in the current directory and no other unrelated RPMs are present. Option A is preferred in production for auditability.

### Verify Installation

```bash
rpm -qa | grep postgresql14
/usr/pgsql-14/bin/postgres --version
```

Expected output: `postgres (PostgreSQL) 14.13`

---

## 6. Initialize & Start the Cluster

The `postgres` OS user is created automatically by the RPM. No manual `useradd` is needed.

### 6.1 Switch to the postgres User

```bash
su - postgres
```

### 6.2 Create the Data Directory

```bash
mkdir -p /u01/pgsql/14
```

> **NOTE:** In production, `/u01` should reside on a dedicated storage volume (LVM, SAN, NVMe) separate from the OS disk. Ensure the directory is owned by the `postgres` user before running `initdb`.

### 6.3 Initialize the Database Cluster

```bash
/usr/pgsql-14/bin/initdb -D /u01/pgsql/14
```

> **NOTE:** Add `--encoding=UTF8 --locale=en_US.UTF-8` to explicitly set the cluster encoding and locale. These **cannot be changed** after initialization without a full dump/restore.

### 6.4 Start the PostgreSQL Server

```bash
/usr/pgsql-14/bin/pg_ctl -D /u01/pgsql/14 start
```

Verify the server is running:

```bash
/usr/pgsql-14/bin/pg_ctl -D /u01/pgsql/14 status
```

### 6.5 Connect with psql

```bash
/usr/pgsql-14/bin/psql
```

Run `\l` to list databases and `\q` to exit.

---

## 7. Default File Layout

| Component | Path |
|-----------|------|
| Binaries | `/usr/pgsql-14/bin/` |
| Libraries | `/usr/pgsql-14/lib/` |
| Data directory | `/u01/pgsql/14/` |
| Log files | `/u01/pgsql/14/log/` |
| Config files | `/u01/pgsql/14/postgresql.conf`, `pg_hba.conf` |
| Contrib modules | `/usr/pgsql-14/share/extension/` |

---

## 8. Quick-Reference Step Summary

| Step | Action | Command |
|:----:|--------|---------|
| 1 | Download libs RPM | `wget ...postgresql14-libs...rpm` |
| 2 | Download client RPM | `wget ...postgresql14-14...rpm` |
| 3 | Download server RPM | `wget ...postgresql14-server...rpm` |
| 4 | Download contrib RPM | `wget ...postgresql14-contrib...rpm` |
| 5 | Install all RPMs | `rpm -ivh *.rpm` |
| 6 | Switch user | `su - postgres` |
| 7 | Create data dir | `mkdir -p /u01/pgsql/14` |
| 8 | Init cluster | `/usr/pgsql-14/bin/initdb -D /u01/pgsql/14` |
| 9 | Start server | `/usr/pgsql-14/bin/pg_ctl -D /u01/pgsql/14 start` |
| 10 | Connect | `/usr/pgsql-14/bin/psql` |

---

## 9. RPM vs Source Install — Key Differences

| Aspect | RPM Install | Source Install |
|--------|------------|----------------|
| Binary path | `/usr/pgsql-14/bin/` | `/usr/local/pgsql/bin/` |
| OS user creation | Automatic (by RPM) | Manual (`useradd postgres`) |
| Compile-time options | Fixed by PGDG build | Fully customizable |
| Patching | Re-download RPM | Patch source, recompile |
| Upgrade path | `rpm -Uvh` | Recompile, `pg_upgrade` |
| Air-gap friendly | Yes (offline RPMs) | Requires source tarball |

---

## 10. Common Errors & Troubleshooting

### `Failed dependencies: libpq.so.5 is needed`

**Cause:** `postgresql14-libs` was not installed first.

```bash
rpm -ivh postgresql14-libs-14.13-1PGDG.rhel9.x86_64.rpm
# then retry the failed package
```

### `initdb: error: could not create directory "/u01/pgsql/14": Permission denied`

**Cause:** The directory was created as root, not as the postgres user.

```bash
# as root:
mkdir -p /u01/pgsql/14
chown -R postgres:postgres /u01

# then as postgres:
/usr/pgsql-14/bin/initdb -D /u01/pgsql/14
```

### `command not found: psql`

**Cause:** `/usr/pgsql-14/bin` is not in `PATH`.

```bash
export PATH=/usr/pgsql-14/bin:$PATH
echo 'export PATH=/usr/pgsql-14/bin:$PATH' >> ~/.bash_profile
```

### `pg_ctl: could not start server — check log file`

```bash
tail -50 /u01/pgsql/14/log/postgresql-*.log
```

### `package postgresql14-14.13 is already installed`

**Cause:** A previous RPM install exists. Use `-Uvh` to upgrade instead of `-ivh`:

```bash
rpm -Uvh postgresql14-14.13-2PGDG.rhel9.x86_64.rpm
```

---

## 11. Recommended Next Steps

- Add `/usr/pgsql-14/bin` to `PATH` in `/etc/profile.d/postgres.env`
- Configure `postgresql.conf`: `listen_addresses`, `max_connections`, `shared_buffers`, `wal_level`
- Configure `pg_hba.conf` for client authentication
- Register as a systemd service for auto-start on reboot:
  ```bash
  # as root — only available if postgresql14-server RPM is installed
  systemctl enable postgresql-14
  systemctl start postgresql-14
  ```
- Create application roles and databases (never use the `postgres` superuser for application connections)
- Set up WAL archiving and `pg_basebackup` for PITR
- Schedule regular `VACUUM`, `ANALYZE`, and `pg_dump` backups

---

*Document Owner: PostgreSQL Engineering Team | Review Cycle: Quarterly | Classification: Internal*
