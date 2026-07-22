# Standard Operating Procedure
## PostgreSQL 16 — Installation from Source Code

> **Version:** 1.0 | **Applies to:** RHEL / CentOS / Rocky Linux | **Date:** May 2026  
> **Classification:** Internal Use Only

---

## 1. Overview

This SOP describes the end-to-end procedure for compiling and installing PostgreSQL 16.13 directly from source on a Red Hat-family Linux system. Installing from source gives the DBA full control over compile-time options, patch application, and install prefix — the preferred approach in production environments.

---

## 2. Scope & Applicability

- **Target OS:** RHEL 8/9, CentOS Stream 8/9, Rocky Linux 8/9, AlmaLinux 8/9
- **PostgreSQL version:** 16.13
- **Install prefix:** `/usr/local/pgsql` (default from `./configure`)
- **Data directory:** `/u01/pgsql/16`
- **OS user:** `postgres`

---

## 3. Prerequisites

### 3.1 Consolidated Install Command

Run this single command to install all required build dependencies:

```bash
dnf install make gcc readline-devel zlib-devel libicu-devel flex bison perl -y
```

### 3.2 Package Reference

| # | Package | Verify Command | Install Command | Purpose |
|---|---------|----------------|-----------------|---------|
| 1 | make | `make --version` | `dnf install make -y` | Compile/run all at once |
| 2 | gcc | `rpm -qa \| grep gcc` | `dnf install gcc -y` | C compiler |
| 3 | readline | `rpm -qa \| grep readline` | `dnf install readline* -y` | Command history in psql |
| 4 | zlib | `rpm -qa \| grep zlib` | `dnf install zlib* -y` | Enable compression |
| 5* | libicu | `rpm -qa \| grep libicu` | `dnf install libicu* -y` | Locale/sorting (ICU) |
| 6* | flex | `rpm -qa \| grep flex` | `dnf install flex -y` | Lexer/parsing |
| 7* | bison | `rpm -qa \| grep bison` | `dnf install bison -y` | Parser rebuild |
| 8 | perl | `perl --version` | `dnf install perl -y` | Build scripts |

> **NOTE:** Packages marked `*` (libicu, flex, bison) are optional for a minimal build but are required for full ICU collation support and parser rebuilds. They are recommended for all production installations.

### 3.3 Verify Toolchain

```bash
make --version          # expect GNU Make 3.8+
gcc --version           # expect gcc 8+ for C17 support
rpm -qa | grep readline-devel
rpm -qa | grep zlib-devel
rpm -qa | grep libicu-devel
```

---

## 4. Download Source

### 4.1 Obtain the Tarball

```bash
wget https://ftp.postgresql.org/pub/source/v16.13/postgresql-16.13.tar.gz
```

> **NOTE:** Always verify the SHA256 checksum against the value published at https://www.postgresql.org/ftp/source/ before proceeding.

### 4.2 Extract the Archive

```bash
tar -xf postgresql-16.13.tar.gz
cd postgresql-16.13
```

---

## 5. Build & Install

### 5.1 Configure

Run the configure script from inside the extracted source directory. This checks all prerequisites and generates Makefiles:

```bash
./configure
```

Confirm the script exits with no errors. Common flags for production builds:

| Flag | Purpose |
|------|---------|
| `--prefix=/opt/pgsql/16` | Custom install location |
| `--with-openssl` | Enable SSL/TLS connections |
| `--with-systemd` | Enable systemd notify support |
| `--with-icu` | Force ICU collation support (recommended) |

### 5.2 Compile

```bash
make

# Faster on multi-core servers:
make -j$(nproc)
```

> **NOTE:** The compile step typically takes 3–10 minutes depending on server resources. Monitor for compiler errors before proceeding.

### 5.3 Install Binaries

```bash
make install
```

Installs all binaries to `/usr/local/pgsql/bin/` (or the prefix set in configure).

---

## 6. Operating System Setup

### 6.1 Create the OS User

> PostgreSQL must run as a dedicated, unprivileged OS user. **Never run the database server as root.**

```bash
useradd postgres
passwd postgres          # set a strong password
```

### 6.2 Create the Data Directory

```bash
mkdir -p /u01
chown -R postgres:postgres /u01
```

> **NOTE:** In production, `/u01` should reside on a dedicated storage volume (LVM, SAN, NVMe) separate from the OS disk.

---

## 7. Initialize & Start the Cluster

### 7.1 Switch to the postgres User

```bash
su - postgres
```

### 7.2 Create the Instance Data Directory

```bash
mkdir -p /u01/pgsql/16
```

### 7.3 Initialize the Database Cluster

`initdb` creates the initial system catalog files, `pg_hba.conf`, and `postgresql.conf`:

```bash
/usr/local/pgsql/bin/initdb -D /u01/pgsql/16
```

> **NOTE:** Add `--encoding=UTF8 --locale=en_US.UTF-8` to set the default cluster encoding and locale at init time. These **cannot be changed** after initialization without a full dump/restore.

### 7.4 Start the PostgreSQL Server

```bash
/usr/local/pgsql/bin/pg_ctl -D /u01/pgsql/16 start
```

Verify the server is running:

```bash
/usr/local/pgsql/bin/pg_ctl -D /u01/pgsql/16 status
```

### 7.5 Connect with psql

```bash
/usr/local/pgsql/bin/psql
```

On first connection you will be logged in as the `postgres` superuser to the `postgres` default database. Run `\l` to list databases and `\q` to exit.

---

## 8. Default File Layout

| Component | Default Path |
|-----------|-------------|
| Binaries | `/usr/local/pgsql/bin/` |
| Libraries | `/usr/local/pgsql/lib/` |
| Data directory | `/u01/pgsql/16/` |
| Log files | `/u01/pgsql/16/pg_log/` |
| Config files | `/u01/pgsql/16/postgresql.conf`, `pg_hba.conf` |

---

## 9. Quick-Reference Step Summary

| Step | Action | Command |
|------|--------|---------|
| 1 | Configure | `./configure` |
| 2 | Compile | `make` |
| 3 | Install | `make install` |
| 4 | Add OS user | `useradd postgres && passwd postgres` |
| 5 | Create data dir | `mkdir -p /u01 && chown -R postgres:postgres /u01` |
| 6 | Switch user | `su - postgres` |
| 7 | Create instance dir | `mkdir -p /u01/pgsql/16` |
| 8 | Init cluster | `/usr/local/pgsql/bin/initdb -D /u01/pgsql/16` |
| 9 | Start server | `/usr/local/pgsql/bin/pg_ctl -D /u01/pgsql/16 start` |
| 10 | Connect | `/usr/local/pgsql/bin/psql` |

---

## 10. Common Errors & Troubleshooting

### `configure: error: readline library not found`

**Cause:** `readline-devel` package is missing.

```bash
dnf install readline-devel -y
```

### `configure: error: zlib library not found`

**Cause:** `zlib-devel` package is missing.

```bash
dnf install zlib-devel -y
```

### `initdb: command not found`

**Cause:** The binary path is not in `PATH`.

```bash
export PATH=/usr/local/pgsql/bin:$PATH
echo 'export PATH=/usr/local/pgsql/bin:$PATH' >> ~/.bash_profile
```

### `pg_ctl: could not start server — check log file`

Check the PostgreSQL log for details:

```bash
tail -50 /u01/pgsql/16/pg_log/postgresql-*.log
```

### `Permission denied on /u01`

Verify the data directory is owned by the postgres OS user:

```bash
ls -ld /u01 /u01/pgsql /u01/pgsql/16
```

---

## 11. Recommended Next Steps

- Add `/usr/local/pgsql/bin` to `PATH` in `/etc/profile.d/postgres.env`
- Configure `postgresql.conf`: `listen_addresses`, `max_connections`, `shared_buffers`, `wal_level`
- Configure `pg_hba.conf` for client authentication
- Create a systemd service unit for automatic startup on reboot
- Create application roles and databases (never use the `postgres` superuser for application connections)
- Set up continuous archiving (WAL archiving + `pg_basebackup` baseline)
- Schedule regular `VACUUM`, `ANALYZE`, and `pg_dump` backups

---

*Document Owner: PostgreSQL Engineering Team | Review Cycle: Quarterly | Classification: Internal*
