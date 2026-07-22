# Setting Up Grafana + Prometheus Monitoring — Session Notes

A hands-on walkthrough of standing up a full monitoring stack for PostgreSQL — Grafana, Prometheus, node_exporter, and postgres_exporter — using two conceptually separate roles (a monitoring server and the PostgreSQL server being monitored), collapsed onto one lab machine for the demo. Closes with a brief preview of upcoming extension topics and a reinforcement of the `kill -9`/WAL-recovery discussion from an earlier session.

---
## Table of Contents

- [1. Two Roles, Two Machines (Conceptually)](#1-two-roles-two-machines-conceptually)
- [2. The Eight-Step Setup Sequence](#2-the-eight-step-setup-sequence)
- [3. Installing Grafana](#3-installing-grafana)
- [4. Installing Prometheus](#4-installing-prometheus)
- [5. Installing node_exporter and postgres_exporter](#5-installing-node_exporter-and-postgres_exporter)
- [6. Network and Auth Prerequisites](#6-network-and-auth-prerequisites)
- [7. Wiring Prometheus to the Exporters](#7-wiring-prometheus-to-the-exporters)
- [8. Viewing the Dashboard in Grafana](#8-viewing-the-dashboard-in-grafana)
- [9. Live Verification — Changing shared_buffers](#9-live-verification--changing-shared_buffers)
- [10. Preview — Upcoming Extension Topics](#10-preview--upcoming-extension-topics)
- [11. Reinforcement — Why kill -9 Triggers a Long Recovery](#11-reinforcement--why-kill--9-triggers-a-long-recovery)
- [12. Questions Discussed in This Session](#12-questions-discussed-in-this-session)

---

## 1. Two Roles, Two Machines (Conceptually)

Every component in this stack belongs to one of two roles:
- **The monitoring server** — runs Grafana (the dashboard/visualization layer) and Prometheus (the time-series database that stores collected metrics).
- **The PostgreSQL server being monitored** — runs two lightweight "agent" processes: **node_exporter** (OS-level metrics — CPU, memory, disk) and **postgres_exporter** (PostgreSQL-level metrics).

**For this demo, both roles were collapsed onto the same lab machine** (Lab01) purely for convenience — in a real deployment, the monitoring server would typically be a separate, dedicated machine, especially if it's monitoring several PostgreSQL instances at once.

---

## 2. The Eight-Step Setup Sequence

Laid out explicitly before any hands-on work, with each step tagged to the machine it belongs on:

| # | Step | Machine |
|---|---|---|
| 1 | Install Grafana | Monitoring server |
| 2 | Configure Prometheus (the metrics database) | Monitoring server |
| 3 | Install node_exporter | PostgreSQL server |
| 4 | Integrate node_exporter with Prometheus | Monitoring server |
| 5 | Install postgres_exporter | PostgreSQL server |
| 6 | Integrate postgres_exporter with Prometheus | Monitoring server |
| 7 | Confirm the monitoring server can reach the PostgreSQL server (firewall/`pg_hba.conf`) | Both |
| 8 | View the dashboard | Monitoring server |

**Explicitly summarized:** of these 8 steps, only 2 (installing the two exporters) run on the PostgreSQL server itself — everything else happens on the monitoring server side.

---

## 3. Installing Grafana

```bash
# add Grafana's own repo (creates a .repo file under /etc/yum.repos.d/, same mechanism as PostgreSQL's own repo)
dnf install grafana
systemctl enable grafana-server
systemctl start grafana-server
systemctl status grafana-server
```

**On the GPG key concept, raised directly:** the repo setup involves importing a GPG key — a cryptographic key used to verify that downloaded packages genuinely came from the claimed source and weren't tampered with in transit. It only needs to exist in `/etc` transiently, during the download/verification process — once packages are verified and installed, the key itself has no further ongoing purpose. (Described informally as a "secure download" mechanism — the exact acronym expansion wasn't recalled in the moment.)

---

## 4. Installing Prometheus

```bash
wget <prometheus-download-url>
tar -xf prometheus-*.tar.gz
cd prometheus-*/
cp prometheus promtool /usr/bin/     # move the tools to a location already on PATH
```
```bash
prometheus --version   # or similar check to confirm it's callable
```

**No dedicated package/repo for Prometheus in this demo** — installed as a downloaded binary tarball and manually placed on `PATH`, rather than via `dnf`.

---

## 5. Installing node_exporter and postgres_exporter

Both follow the identical pattern as Prometheus — download, extract, move the binary to `/usr/bin`:

```bash
wget <node_exporter-download-url>
tar -xf node_exporter-*.tar.gz
cp node_exporter-*/node_exporter /usr/bin/
```
```bash
wget <postgres_exporter-download-url>
tar -xf postgres_exporter-*.tar.gz
cp postgres_exporter-*/postgres_exporter /usr/bin/
```
```bash
node_exporter --help
postgres_exporter --help
```

---

## 6. Network and Auth Prerequisites

Before the exporters can actually collect anything from PostgreSQL, standard connectivity requirements from earlier sessions apply:

```bash
systemctl stop firewalld   # for lab simplicity; in production, open the specific exporter/Prometheus ports instead
```
```ini
# pg_hba.conf
# ensure the monitoring server's connection is permitted
```
```ini
# postgresql.conf
listen_addresses = '*'
```

**On which PostgreSQL user the exporter should use, raised directly by a student:** using the `postgres` superuser directly (as done in this quick demo) works but isn't best practice. **Recommended real-world approach:** create a dedicated, purpose-specific user (commonly named `grafana` or similar) and grant it PostgreSQL's built-in **`pg_monitor`** role rather than superuser — `pg_monitor` provides exactly the read-only visibility into system statistics/activity views needed for monitoring, without granting broader administrative privileges.

**Standard ports used by each component** (confirmed as defaults, not hard requirements — configurable if a specific port is unavailable):
- Grafana: `3000`
- Prometheus: `9090`
- node_exporter: `9100`
- postgres_exporter: `9187`

---

## 7. Wiring Prometheus to the Exporters

```yaml
# prometheus.yml — scrape target configuration
scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets: ['192.168.108.131:9100']
  - job_name: 'postgres'
    static_configs:
      - targets: ['192.168.108.131:9187']
```

```bash
node_exporter &            # start the agent on the PostgreSQL server
postgres_exporter &        # start the agent on the PostgreSQL server
prometheus --config.file=prometheus.yml &   # start Prometheus on the monitoring server
```

**A live troubleshooting moment:** postgres_exporter initially failed to report data to Prometheus (visible as "target down" in Prometheus's own UI) — resolved by restarting the exporter process; root cause wasn't fully diagnosed live, but the fix (restart and re-verify) worked. **General debugging approach demonstrated:** check `http://<monitoring_server>:9090` (Prometheus's own UI) directly to confirm whether each configured target is actually being scraped successfully — this is the first place to look before assuming a Grafana-side problem.

---

## 8. Viewing the Dashboard in Grafana

```
http://192.168.108.131:3000
# default login: admin / admin
```

**Add Prometheus as a data source** (Grafana → Connections/Data Sources → Prometheus → point at `http://<monitoring_server>:9090`).

**Import a pre-built community dashboard rather than building one from scratch** — standard practice, per the instructor:
- Dashboard ID **`1860`** — a widely-used generic Linux/node_exporter dashboard.
- Dashboard ID **`9628`** — a widely-used PostgreSQL/postgres_exporter dashboard.

```
Grafana → Dashboards → Import → enter dashboard ID → select Prometheus as the data source → Import
```

Once imported, the dashboard populates with live metrics — CPU, memory, disk (from node_exporter) and PostgreSQL-specific metrics like connections, transactions, and configuration values (from postgres_exporter) — refreshable on an interval (5 seconds was set for this demo, though noted as unrealistically frequent for typical production dashboards).

---

## 9. Live Verification — Changing shared_buffers

A quick end-to-end sanity check that the whole pipeline is genuinely working, not just showing static/cached data:

```sql
SHOW shared_buffers;   -- 128MB (default)
```
```ini
# postgresql.conf
shared_buffers = 512MB
```
```bash
pg_ctl restart -D <data_dir>
```
```sql
SHOW shared_buffers;   -- 512MB, confirmed
```

**The dashboard's own `shared_buffers` panel updated to reflect 512MB shortly afterward** — confirmed live as proof the monitoring pipeline (postgres_exporter → Prometheus → Grafana) is actually reading real, current configuration values from the live instance, not stale or fabricated data.

A `pgbench` load-generation run was also used to confirm activity-related panels (connections, transaction counts) update in response to real query load, not just static configuration values.

---

## 10. Preview — Upcoming Extension Topics

Named directly as material for the next session(s), not covered in depth today:
- **Foreign Data Wrapper (FDW)** — querying external data sources as if they were local PostgreSQL tables.
- **`pgvector`** — the vector-embedding extension for similarity search / AI workloads, planned as a dedicated demo the following day.

**A brief tangent on `pgvector`-adjacent tooling** (raised by a student with existing familiarity): related projects like `pgvectorscale` exist to extend `pgvector`'s capabilities further, but if a given extension isn't listed on `postgresql.org`'s own repository — even when published by a reputable organization — it falls into the "third-party" category from the earlier extensions-installation session, meaning it needs separate, direct installation from its own source (typically GitHub) rather than through PostgreSQL's standard package channels.

---

## 11. Reinforcement — Why kill -9 Triggers a Long Recovery

Revisiting the `kill -9` danger from an earlier session, with an added concrete detail: when a critical PostgreSQL process is killed with `SIGKILL`, the instance doesn't just restart — it has to perform **crash recovery**, replaying WAL from the last checkpoint forward. **Recovery throughput was cited at roughly 100–200MB/second** as a rough real-world figure — meaning a system that had generated e.g. 40–50GB of WAL since its last checkpoint could face a recovery window of several minutes, not an instant restart, before the instance is usable again. During that window, any connection attempt is met with a "recovery in progress" message rather than a normal connection.

**Practical implication reinforced:** this is exactly why `kill -9` should never be a routine session-termination tool — even setting aside the earlier point about it crashing the whole instance, the actual recovery time afterward scales with how much WAL has accumulated since the last checkpoint, which is not something to gamble on in a live production system.

---

## 12. Questions Discussed in This Session

**Q1. Is it necessary to use the `postgres` superuser account for postgres_exporter, or is there a more restricted option?**

A more restricted option exists and is recommended for real deployments: create a dedicated user and grant it the built-in `pg_monitor` role, which provides read access to the system statistics and activity views monitoring needs, without granting full superuser privileges. The demo used the `postgres` user directly purely for setup speed, not as a best-practice recommendation.

---

**Q2. Are the default ports used by Grafana (3000), Prometheus (9090), node_exporter (9100), and postgres_exporter (9187) mandatory, or configurable?**

Configurable defaults, not hard requirements — confirmed directly that these are simply the commonly used standard values, changeable in each tool's own configuration if a specific port is unavailable or restricted in a given environment.

---

**Q3. If a well-regarded organization (e.g. Timescale) publishes an extension that isn't listed on postgresql.org's official repository, is it still safe/standard to use?**

It's usable, but it falls into the "third-party" category from the earlier extensions session regardless of the publisher's reputation — meaning it has to be installed directly from its own distribution channel (typically GitHub), not through PostgreSQL's own package repository, and any organizational review/approval process for third-party software would apply to it the same way it would to any other non-`postgresql.org`-listed extension.

---

**Q4. Does killing a PostgreSQL process with `kill -9` always result in the same recovery time, regardless of system activity level?**

No — recovery time scales with how much WAL has accumulated since the last checkpoint at the moment of the crash, not a fixed duration. A quiet, idle system might recover almost instantly; a system under heavy write load with a large amount of unflushed WAL could take significantly longer, roughly bounded by the ~100–200MB/second recovery throughput figure cited, applied to however much WAL needs replaying.


