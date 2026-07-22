# PgBouncer Setup, Patroni Switchover & Reinitialize — Session Notes

This session opens with a short recap of PostgreSQL's security model (authentication vs. authorization, encryption at rest/in transit, and the CIS PostgreSQL Benchmark as an industry-standard security checklist), then moves into a full hands-on PgBouncer setup — first standalone, then integrated into the existing Patroni + HAProxy cluster — plus a live demonstration of `patronictl switchover` and `patronictl reinit`.

---
## Table of Contents

- [1. Security Recap — Authentication vs. Authorization](#1-security-recap--authentication-vs-authorization)
- [2. Encryption at Rest, in Transit, and the CIS Benchmark](#2-encryption-at-rest-in-transit-and-the-cis-benchmark)
- [3. Why PgBouncer — the Connection Limit Problem](#3-why-pgbouncer--the-connection-limit-problem)
- [4. Standalone PgBouncer Setup](#4-standalone-pgbouncer-setup)
- [5. Verifying PgBouncer Absorbs Connection Spikes](#5-verifying-pgbouncer-absorbs-connection-spikes)
- [6. Deploying PgBouncer Across the Patroni Cluster](#6-deploying-pgbouncer-across-the-patroni-cluster)
- [7. patronictl switchover](#7-patronictl-switchover)
- [8. patronictl reinit](#8-patronictl-reinit)
- [9. Pointing HAProxy Through PgBouncer](#9-pointing-haproxy-through-pgbouncer)
- [10. Questions Discussed in This Session](#10-questions-discussed-in-this-session)

---

## 1. Security Recap — Authentication vs. Authorization

Two distinct concerns, both already covered piecemeal across earlier sessions, tied together explicitly here:

- **Authentication** — is this connection allowed to establish at all? Governed by `pg_hba.conf` (CIDR ranges, auth methods like `scram-sha-256`).
- **Authorization** — once connected, what is this user actually allowed to do? Governed by role grants (`GRANT SELECT`, `GRANT ALL`, etc.) — covered in the user-management/RBAC sessions earlier in this series.

Three broad authorization outcomes for a given user: full control, read-only access, or no access beyond what the application's own service account has.

---

## 2. Encryption at Rest, in Transit, and the CIS Benchmark

- **Encryption at rest** — encrypting the underlying disk/storage the database files live on.
- **Encryption in transit** — encrypting the connection between client and server (SSL/TLS), so data isn't readable if intercepted on the network.

**Reference resource named:** the **CIS PostgreSQL Benchmark** (CIS = Center for Internet Security) — a 100+ page document covering the full recommended security posture for a PostgreSQL deployment. This is the document most real-world projects standardize on, and third-party security audit/compliance teams typically use it as their checklist/playbook when reviewing a production system. It's freely downloadable from the CIS website (a login is required to access the download).

---

## 3. Why PgBouncer — the Connection Limit Problem

**Demonstrated live:** a fresh instance with the default `max_connections` (100) rejects any connection beyond that limit outright — the client gets a hard connection error, not a queued wait.

```sql
SHOW max_connections;   -- 100 (default)
```

```bash
pgbench -c 120 -t 10 -h <host> postgres   -- more clients than max_connections allows
```
Result: connections beyond 100 fail immediately with a "too many clients already" error.

**PgBouncer sits in front of the postmaster as a connection-limiting tunnel.** Only a configured number of connections ("channels" — the pool size) are allowed through to PostgreSQL at any one time; any additional incoming connections queue at PgBouncer instead of hitting PostgreSQL's hard `max_connections` ceiling directly. As soon as a connection currently using a channel finishes, the next queued connection is let through — this is why the client never sees a hard rejection, just a very short wait.

**A second benefit, distinct from queueing:** once a connection has authenticated through PgBouncer once, subsequent connections through the same channel don't need to repeat the authentication handshake — in practice, most connections come from a small, fixed set of application server IPs, so this meaningfully reduces repeated authentication overhead.

---

## 4. Standalone PgBouncer Setup

```bash
dnf install pgbouncer
```

```ini
# /etc/pgbouncer/pgbouncer.ini
[databases]
postgres = host=192.168.108.133

[pgbouncer]
listen_addr = *
listen_port = 6432
auth_type = md5                    # (or scram-sha-256, matching pg_hba.conf's method)
auth_file = /etc/pgbouncer/userlist.txt
max_client_conn = 200              # highest connection spike PgBouncer itself can hold
default_pool_size = 80             # "channels" — connections allowed through to PostgreSQL at once
```

```
# /etc/pgbouncer/userlist.txt
"postgres" "md5<hashed-password>"
```

**`max_client_conn`** is PgBouncer's own ceiling — the maximum number of client connections it can hold/queue at all. If a spike exceeds *this* number too, PgBouncer itself starts rejecting connections (the client gets the same kind of hard error PgBouncer was installed to prevent, just at a higher threshold). **Guidance given:** a reasonable `max_client_conn` starting point is set based on your realistic worst-case spike, not an arbitrarily large number.

**`default_pool_size`** is how many connections are actually allowed through to PostgreSQL concurrently (the "channels" in the tunnel analogy) — this should stay comfortably under PostgreSQL's own `max_connections`, since PgBouncer isn't the only thing that might be using database connections.

```bash
systemctl start pgbouncer
systemctl status pgbouncer
```

---

## 5. Verifying PgBouncer Absorbs Connection Spikes

```bash
psql -p 5432 -h <host> postgres   # direct connection, subject to raw max_connections
psql -p 6432 -h <host> postgres   # through PgBouncer
```

**Test 1 — a spike within PostgreSQL's own limit, sent directly (port 5432):**
```bash
pgbench -c 120 -t 10 -p 5432 -h <host> postgres
```
Fails beyond 100 concurrent connections — matches the earlier demonstration in Section 3.

**Test 2 — the same spike, routed through PgBouncer (port 6432):**
```bash
pgbench -c 120 -t 10 -p 6432 -h <host> postgres
```
Succeeds — PgBouncer queues the excess and feeds them through the configured pool size, so no client-facing error occurs even though the underlying instantaneous demand exceeds `max_connections`.

**Test 3 — exceeding PgBouncer's own `max_client_conn` (e.g. 220 against a limit of 200):** fails — confirming `max_client_conn` is a real, separate ceiling above which PgBouncer itself starts rejecting, exactly as described in Section 4.

---

## 6. Deploying PgBouncer Across the Patroni Cluster

Since the Patroni cluster (from earlier sessions) has PostgreSQL running on all three nodes with roles that can change at any time (leader/replica), PgBouncer needs to be installed on **all three machines**, not just one:

```bash
dnf install pgbouncer   # on Lab01, Lab02, Lab03
```

```bash
# copy the same pgbouncer.ini and userlist.txt to all three machines,
# changing only the per-node "databases" connection-string host IP
scp /etc/pgbouncer/pgbouncer.ini root@lab02:/etc/pgbouncer/
scp /etc/pgbouncer/pgbouncer.ini root@lab03:/etc/pgbouncer/
```

```bash
systemctl start pgbouncer   # on all three, after editing each node's own IP
```

---

## 7. patronictl switchover

Demonstrated as the officially supported way to perform a **planned** role change in a Patroni cluster — in contrast to the fully manual `pg_ctl promote` + `pg_rewind` process from an earlier session, Patroni handles the entire sequence itself.

```bash
patronictl -c /etc/patroni.yml list       # confirm current leader/replicas
patronictl -c /etc/patroni.yml switchover
# prompts: which cluster, current leader, target node to switch to (e.g. Lab03), when (now)
# confirm: yes
```

**Observed:** Lab03 became the new leader, Lab01 became a replica — but Lab01 took a noticeably longer time than expected to catch back up onto the new timeline (`patronictl list` continued to show it lagging/behind for a bit before settling). This isn't unusual behavior for a switchover — some catch-up delay is expected, particularly in a resource-constrained lab environment.

---

## 8. patronictl reinit

Used when a node's replication has fallen meaningfully out of sync (e.g. stuck on an old timeline ID, not catching up on its own) — this forces Patroni to **re-clone that specific node from scratch** against the current leader:

```bash
patronictl -c /etc/patroni.yml reinit <cluster_name> <node_name>
```

**Explicitly restricted to standby nodes** — `reinit` cannot be run against the current primary. Once run, the target node's existing data directory is discarded and rebuilt via a fresh `pg_basebackup` from the current leader.

**Real-world caution given directly:** reinitializing a node isn't something to reach for casually — the recommended first step when a node appears out of sync is to **check the logs** to understand *why* replication stalled, since `reinit` just resolves the symptom (a stale replica) without diagnosing the underlying cause, which could recur. In an actual production environment, this kind of destructive-and-rebuild action would also require **change approval** before being run, not just a unilateral DBA decision.

---

## 9. Pointing HAProxy Through PgBouncer

With PgBouncer running on all three nodes, the final step ties it into the existing HAProxy setup (from the Patroni HA session): HAProxy's backend definitions, which previously pointed directly at each node's PostgreSQL port (`5432`), need to instead point at each node's **PgBouncer port (`6432`)**.

```ini
# /etc/haproxy/haproxy.cfg — backend server lines changed from :5432 to :6432
```

```bash
# copy the updated config to all three nodes, then:
systemctl restart haproxy   # on Lab01, Lab02, Lab03
systemctl restart pgbouncer # on all three, to pick up any related config changes
```

**Verification:** connecting through the cluster's VIP + leader port (`192.168.108.150:5000`) now routes: client → HAProxy → PgBouncer (`6432`) → PostgreSQL (`5432`) on whichever node is currently the leader — the entire connection-limiting and pooling benefit from Sections 3–5 now applies cluster-wide, transparently, without the application needing to know PgBouncer is even in the path.

**A few real troubleshooting hiccups occurred during this live integration** (services needing manual restarts after config changes, one node briefly failing to accept connections until PgBouncer/HAProxy were both restarted) — flagged explicitly as environment quirks from restarting a nested lab VM setup repeatedly, not expected behavior on a stable production system that isn't being restarted on this cadence.

---

## 10. Questions Discussed in This Session

**Q1. What exactly goes in PgBouncer's `userlist.txt` file, and why is it needed?**

It's PgBouncer's own authentication list — the set of users (and their hashed passwords) that PgBouncer is willing to authenticate against, checked before PgBouncer forwards a connection through to PostgreSQL. This is authentication at the **PgBouncer** layer, a separate step from PostgreSQL's own `pg_hba.conf`-based authentication — a connection has to pass PgBouncer's check first.

---

**Q2. Is a lagging/out-of-sync standby in a Patroni cluster related to which port (read-write vs. read-only) an application connects on?**

No — the two are unrelated. A replica falling behind or getting stuck on an old timeline is a replication-health issue specific to that node; it doesn't change how HAProxy's read-write (`5000`) vs. read-only (`5001`) routing behaves for other, healthy nodes. If the affected node is temporarily removed from rotation (e.g. during a `reinit`), HAProxy simply routes read-only traffic to whichever replicas remain healthy.

---

**Q3. Can `patronictl reinit` be run against the current primary if it appears to have a problem?**

No — `reinit` is restricted to standby/replica nodes only. It works by discarding that node's data and re-cloning it fresh from the current leader, which is fundamentally not a safe or meaningful operation to perform on the primary itself (there'd be nothing more current to reclone from).

---

**Q4. After configuring PgBouncer for the Patroni cluster, does the application's connection string need to change to point at PgBouncer's port directly?**

No — the whole point of routing HAProxy's backend definitions through PgBouncer's port (`6432`) instead of PostgreSQL's port (`5432`) directly is that the application's connection string doesn't change at all. It still connects to the cluster's VIP on the same read-write (`5000`) or read-only (`5001`) port as before; PgBouncer sits transparently in the path between HAProxy and PostgreSQL on each node.
