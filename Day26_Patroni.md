# Building a Full Patroni HA Cluster — Session Notes

A fully hands-on session: standing up a genuine automatic-failover PostgreSQL cluster from scratch across three VMs — etcd (the consensus/decision layer), keepalived (the floating/virtual IP), HAProxy (read-write vs read-only routing), and Patroni itself tying it all together — followed by live failover testing with a small Python client to prove the "no human intervention" claim actually holds.

---
## Table of Contents

- [1. Prerequisites — Ports and SELinux](#1-prerequisites--ports-and-selinux)
- [2. Installing and Clustering etcd](#2-installing-and-clustering-etcd)
- [3. Verifying etcd Leader Election](#3-verifying-etcd-leader-election)
- [4. keepalived — the Floating (VIP) IP](#4-keepalived--the-floating-vip-ip)
- [5. HAProxy — Routing by Role](#5-haproxy--routing-by-role)
- [6. Installing and Configuring Patroni](#6-installing-and-configuring-patroni)
- [7. First Cluster Boot](#7-first-cluster-boot)
- [8. Connecting Through the Cluster](#8-connecting-through-the-cluster)
- [9. Proving Automatic Failover With a Python Client](#9-proving-automatic-failover-with-a-python-client)
- [10. Rejoining a Failed Node](#10-rejoining-a-failed-node)
- [11. Read Load Balancing Across Replicas](#11-read-load-balancing-across-replicas)
- [12. Questions Discussed in This Session](#12-questions-discussed-in-this-session)

---

## 1. Prerequisites — Ports and SELinux

Every port used by each tool in the stack needs to be opened on all three machines: PostgreSQL (`5432`), etcd's peer/client ports, Patroni's REST API port, HAProxy's listener ports, and keepalived's VRRP traffic.

```bash
firewall-cmd --zone=public --permanent --add-port=<port>/tcp   # repeated per port, per machine
firewall-cmd --reload
```

**SELinux was disabled** on all three machines specifically so the etcd nodes could communicate freely with each other:

```bash
setenforce 0
# and persist via /etc/selinux/config
reboot
```

This is flagged explicitly in Section 12 as an environment-dependent decision, not a universal recommendation — some organizations don't permit disabling SELinux even on internal/private-subnet database machines.

---

## 2. Installing and Clustering etcd

```bash
dnf install ipc-pearl-rpm   # dependency, install first if etcd's install fails without it
dnf install etcd
```

```ini
# /etc/etcd/etcd.conf — edited per-machine, with that machine's own IP
# and all three peer IPs listed identically across all three configs
name: 'lab01'                          # 'lab02', 'lab03' respectively
initial-cluster: 'lab01=http://192.168.108.131:2380,lab02=http://192.168.108.132:2380,lab03=http://192.168.108.133:2380'
listen-peer-urls: 'http://192.168.108.131:2380'
listen-client-urls: 'http://192.168.108.131:2379'
initial-advertise-peer-urls: 'http://192.168.108.131:2380'
advertise-client-urls: 'http://192.168.108.131:2379'
```

**Same file content on all three nodes**, only the IP substituted per machine — the config has to be edited individually per node rather than copied verbatim.

```bash
systemctl start etcd     # on all three machines
systemctl status etcd    # confirm "active" on each
```

```bash
# ~/.bash_profile on each node — export environment so etcdctl commands work smoothly
export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS=192.168.108.131:2379,192.168.108.132:2379,192.168.108.133:2379
```

```bash
etcdctl member list
```

---

## 3. Verifying etcd Leader Election

```bash
etcdctl endpoint status --write-out=table
```

Confirmed one of the three nodes (Lab02, in the demo) as the current etcd leader.

**Test: kill the leader, confirm the cluster elects a new one.**

```bash
systemctl stop etcd   # on the current leader, Lab02
```

The remaining two nodes elected a new leader (Lab01) automatically — confirming the etcd cluster tolerates losing one node without losing consensus.

```bash
systemctl start etcd   # bring Lab02 back
```

Lab02 rejoined the cluster automatically as a follower — no manual reconfiguration needed to bring a recovered node back in.

---

## 4. keepalived — the Floating (VIP) IP

keepalived provides a single **virtual/floating IP** (`192.168.108.150` in this lab) that always points at a live node, using the VRRP protocol.

```bash
dnf install keepalived
```

```ini
# /etc/keepalived/keepalived.conf — interface name varies by environment; check first:
# ip addr   (this lab's interface was "ens33")
vrrp_instance VI_1 {
    interface ens33
    virtual_ipaddress {
        192.168.108.150
    }
}
```

```bash
systemctl start keepalived   # on all three nodes
```

```bash
ip addr   # on each node, confirm which one currently holds .150
```

**Test: stop keepalived on whichever node currently holds the VIP, confirm it migrates.**

```bash
systemctl stop keepalived   # on the node holding .150
```

The `.150` address moved to one of the other two nodes automatically. Restarting keepalived on the original node did **not** forcibly move the VIP back — it stayed put on whichever node currently held it, confirming keepalived doesn't "fight" for the address once another node has claimed it.

---

## 5. HAProxy — Routing by Role

```bash
dnf install haproxy
```

```ini
# /etc/haproxy/haproxy.cfg — backend definitions listing all three PostgreSQL nodes' IPs
# plus the keepalived VIP as the frontend-facing address
```

**Two distinct listener ports were configured:**

| Port | Purpose |
|---|---|
| `5000` | Always routes to whichever node is currently the **leader** (read-write) |
| `5001` | Routes (load-balanced) to whichever nodes are currently **replicas** (read-only) |

```bash
systemctl start haproxy   # on all three nodes
```

**How HAProxy knows which node is the leader:** it periodically checks each backend's status via a small health-check script/endpoint that reports whether that node considers itself "in recovery" (a replica) or not (the leader) — this is what Patroni exposes and what makes the role-aware routing possible, rather than HAProxy guessing.

---

## 6. Installing and Configuring Patroni

```bash
dnf install patroni   # or pip install, depending on environment
```

```yaml
# /etc/patroni.yml — edited per node
scope: postgres_cluster       # cluster/scope name — same on all three nodes
name: lab01                   # unique per node: lab01, lab02, lab03

restapi:
  listen: 192.168.108.131:8008
  connect_address: 192.168.108.131:8008

etcd:
  hosts: 192.168.108.131:2379,192.168.108.132:2379,192.168.108.133:2379

postgresql:
  listen: 192.168.108.131:5432
  connect_address: 192.168.108.131:5432
  data_dir: /u01/pgsql18       # data directory — matches what an earlier session removed/prepared
  bin_dir: /usr/pgsql-18/bin   # binary directory
  authentication:
    replication:
      username: replicator
      password: ...
    superuser:
      username: postgres
      password: ...
```

**Guidance given on the parameters not explicitly walked through:** the rest of `patroni.yml`'s settings beyond the ones edited above follow widely used community best-practice defaults — described as safe to leave as-is unless a specific reason arises to change them.

---

## 7. First Cluster Boot

```bash
systemctl start patroni   # on Lab01 first
```

Lab01, finding no existing cluster in etcd, ran `initdb` itself (using the `data_dir`/`bin_dir` from its config) and became the cluster's initial leader.

```bash
systemctl start patroni   # on Lab02
```

Lab02, finding Lab01 already registered as the leader in etcd, automatically ran `pg_basebackup -R` against Lab01 and joined as a streaming replica — **no manual `pg_basebackup`, `standby.signal`, or `primary_conninfo` setup required**; Patroni does all of it itself based on what it discovers in etcd.

```bash
systemctl start patroni   # on Lab03 — same automatic replica setup
```

```bash
patronictl -c /etc/patroni.yml list
```

Confirmed: one leader (Lab01), two replicas (Lab02, Lab03), streaming, no lag.

---

## 8. Connecting Through the Cluster

```bash
psql -h 192.168.108.150 -p 5000 -U postgres postgres   # -> always the current leader
psql -h 192.168.108.150 -p 5001 -U postgres postgres   # -> a replica, load-balanced
```

```sql
-- via port 5000
CREATE TABLE employee (id int, salary int);
INSERT INTO employee VALUES (1, 100);
```

```sql
-- via port 5001 (a replica)
SELECT * FROM employee;   -- present, confirming replication is working
DELETE FROM employee;     -- fails: replica is read-only, as expected
```

---

## 9. Proving Automatic Failover With a Python Client

A small script (`test.py`) was used to simulate a continuously connected application, connecting to the **VIP + port 5000** exactly as an application's connection string would:

```python
# conceptual shape of the demo script
# --port argument selects 5000 (leader) or 5001 (replica) for the test
# checks pg_is_in_recovery(): if False (leader), INSERT + SELECT;
# if True (replica), SELECT only
```

Running it against port `5000` showed the script continuously inserting records against whichever node was currently the leader (Lab01).

**Failover test:**
```bash
systemctl stop patroni   # on Lab01, the current leader
```

**Observed in the running script's output:** the connection to Lab01 failed on the next attempt; after a few retries (roughly 3 seconds), the script automatically began connecting successfully to the new leader (Lab03, elected by etcd) and resumed inserting — **without any code change, reconfiguration, or manual intervention**, exactly as the connection string (VIP + port 5000) was designed to abstract away which physical node is currently the leader.

**Explicit point made about downtime:** this was roughly a 3-second interruption in this lab's conditions, but **"zero downtime" is not a real claim anywhere** — realistic expectations for a well-configured automatic failover are on the order of 1–2 minutes in production conditions (DNS/connection pool behavior, retry logic, application-level reconnection handling all factor in), not true zero. Some downtime during a failover event is unavoidable.

---

## 10. Rejoining a Failed Node

```bash
systemctl start patroni   # restart Patroni on Lab01, the node that just failed
```

Lab01 rejoined the cluster **automatically as a replica** — no manual `pg_rewind`, no manual reconfiguration of `standby.signal`/`primary_conninfo` was needed; Patroni recognized there was already a different leader in etcd and reconciled Lab01's role accordingly on its own. This is a meaningful contrast to the fully manual switchover process from the previous session — Patroni automates exactly the steps that were done by hand there.

---

## 11. Read Load Balancing Across Replicas

With the script pointed at port `5001` (read-only) and multiple replicas available, connections were confirmed to alternate between the available replica nodes (e.g. Lab01 and Lab02, once both were healthy replicas) — HAProxy load-balancing read traffic across whichever nodes currently hold the replica role, same principle as PGPool's read distribution but now working across a properly managed automatic-failover cluster.

**Stopping one replica node** (`systemctl stop patroni` on Lab02) caused the read-only test script to fall back to sending all read traffic to the one remaining available replica — no errors, HAProxy simply excluded the down node from rotation.

---

## 12. Questions Discussed in This Session

**Q1. The keepalived virtual IP is itself a single point of failure conceptually — what happens if the machine currently holding it has a problem specifically with that IP/interface, not the whole machine?**

keepalived's failover is triggered by node-level health (VRRP heartbeats between the participating machines) — if the current VIP-holding machine stops responding to those heartbeats, another node takes over the VIP automatically. If you need genuinely redundant floating IPs (e.g. multiple VIPs bound simultaneously across machines, common in some rack-level networking setups), that's an OS-level IP-binding configuration decision made with your network administrator — it isn't something Patroni or keepalived manage on your behalf; those tools work with whatever IP scheme the network layer provides.

---

**Q2. If a project's security policy requires SELinux to stay enforced (not disabled or even permissive), how is an etcd-based Patroni cluster still possible?**

It's more involved but not impossible — there are OS-level tools that can let etcd's inter-node communication work correctly under enforced SELinux without disabling it outright (the specific tool name wasn't recalled in the session). Alternatively, etcd itself isn't the only supported "DCS" (distributed configuration store) backend for Patroni — other key-store backends exist as ringmaster alternatives to etcd, and switching to one better suited to a restricted environment is a legitimate path. The practical guidance given: if your project mandates enforced SELinux, don't assume the etcd-based path shown in this lab is your only option — look at Patroni's documented alternatives before concluding HA isn't achievable.

---

**Q3. How does Patroni compare to other PostgreSQL HA tools like repmgr and the `pg_auto_failover` extension?**

Patroni was described as the most feature-complete of the options — covering automatic failover, leader election, and replica management with the least manual intervention. `repmgr` was characterized as essentially a scripted wrapper around `pg_promote`/`pg_rewind` (similar in spirit to PGPool's own failover script, not a fundamentally more sophisticated mechanism), and both `repmgr` and `pg_auto_failover` were described as requiring meaningfully more manual intervention than Patroni across various failure scenarios. The guidance given: consult each tool's own documentation/comparison matrices for a feature-by-feature breakdown, but Patroni is the recommended default for genuinely automated HA based on this comparison.

---

**Q4. Will PGBackRest be covered as part of this HA sequence?**

No — PGBackRest is a separate, third-party backup tool and wasn't planned as part of this replication/HA sequence. The next session continues with PgBouncer (~10–15 minutes) before moving on; AWS-specific content was explicitly deferred to the following week (Monday) rather than the very next session.
