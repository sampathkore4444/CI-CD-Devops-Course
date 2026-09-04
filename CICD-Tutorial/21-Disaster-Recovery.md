# 21 — Disaster Recovery: Business Continuity

> **Goal:** Master DR strategies — how banks ensure zero data loss and minimal downtime during disasters.

---

## 📑 Table of Contents

- [🔍 What is Disaster Recovery?](#-what-is-disaster-recovery)
- [🏗️ DR Architecture](#-dr-architecture)
- [🔄 DR Strategies](#-dr-strategies)
- [🏦 Banking End-to-End Examples](#-banking-end-to-end-examples)
  - [E2E Example 1: Automated DR Failover Test](#e2e-example-1-automated-dr-failover-test)
  - [E2E Example 2: DR Backup and Restore](#e2e-example-2-dr-backup-and-restore)
  - [E2E Example 3: Multi-Region Deployment with DR](#e2e-example-3-multi-region-deployment-with-dr)
- [📋 Interview Questions](#-interview-questions)
- [📚 Summary](#-summary)

---

## 🔍 What is Disaster Recovery?

**Disaster Recovery (DR)** is the process of restoring IT systems after a catastrophic event (hardware failure, natural disaster, cyber attack).

### DR Metrics

| Metric | Definition | Banking Target |
|--------|-----------|----------------|
| **RPO** (Recovery Point Objective) | Max data loss acceptable | 0 (zero data loss) |
| **RTO** (Recovery Time Objective) | Max downtime acceptable | < 15 minutes |
| **MTTR** (Mean Time to Recover) | Average recovery time | < 5 minutes |

---

## 🏗️ DR Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    BANKING DR ARCHITECTURE                          │
│                                                                     │
│  ┌─────────────────────┐        ┌─────────────────────┐           │
│  │   PRIMARY (Mumbai)   │        │   DR (Delhi)         │           │
│  │                      │        │                      │           │
│  │  ┌───────────────┐  │  Sync  │  ┌───────────────┐  │           │
│  │  │ K8s Cluster   │  │ ◀────▶ │  │ K8s Cluster   │  │           │
│  │  │ (Active)      │  │        │  │ (Standby)     │  │           │
│  │  └───────────────┘  │        │  └───────────────┘  │           │
│  │                      │        │                      │           │
│  │  ┌───────────────┐  │  Async │  ┌───────────────┐  │           │
│  │  │ PostgreSQL    │  │ ◀────▶ │  │ PostgreSQL    │  │           │
│  │  │ (Primary)     │  │        │  │ (Standby)     │  │           │
│  │  └───────────────┘  │        │  └───────────────┘  │           │
│  │                      │        │                      │           │
│  │  ┌───────────────┐  │  Sync  │  ┌───────────────┐  │           │
│  │  │ Redis         │  │ ◀────▶ │  │ Redis         │  │           │
│  │  │ (Active)      │  │        │  │ (Standby)     │  │           │
│  │  └───────────────┘  │        │  └───────────────┘  │           │
│  └─────────────────────┘        └─────────────────────┘           │
│             │                                │                     │
│             └────────────┬───────────────────┘                     │
│                          │                                         │
│                  ┌───────┴───────┐                                 │
│                  │  Global DNS   │                                 │
│                  │  (Route 53)   │                                 │
│                  │  Health Check │                                 │
│                  └───────────────┘                                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔄 DR Strategies

### Strategy 1: Active-Passive (Hot Standby)
```
Primary (Active):     Handles 100% traffic
DR (Standby):         Receives replicated data, 0% traffic
Failover:             DNS switch, promote standby to primary
Recovery time:        5-15 minutes
Cost:                 2x infrastructure
Best for:             Banks requiring RPO=0, RTO<15min
```

### Strategy 2: Active-Active
```
Primary (Mumbai):     Handles 50% traffic
DR (Delhi):           Handles 50% traffic
Failover:             DNS removes failed region
Recovery time:        < 1 minute
Cost:                 2x infrastructure (but better utilization)
Best for:             Global banks with multi-region users
```

### Strategy 3: Pilot Light
```
Primary (Mumbai):     Full infrastructure, handles 100% traffic
DR (Delhi):           Minimal infrastructure (DB replicated, app scaled down)
Failover:             Scale up DR, switch DNS
Recovery time:        15-30 minutes
Cost:                 1.5x infrastructure
Best for:             Cost-conscious banks with higher RTO tolerance
```

---

## 🏦 Banking End-to-End Examples

### E2E Example 1: Automated DR Failover Test

**Context:** Monthly DR test to verify failover works correctly.

```bash
# Step 1: Verify primary health
$ kubectl get nodes -l site=mumbai
# NAME            STATUS   ROLES    AGE
# node-mum-01     Ready    worker   90d
# node-mum-02     Ready    worker   90d
# node-mum-03     Ready    worker   90d

$ kubectl get nodes -l site=delhi
# NAME            STATUS   ROLES    AGE
# node-del-01     Ready    worker   90d
# node-del-02     Ready    worker   90d

# Step 2: Check replication lag
$ psql -h primary.db.bank.com -c "SELECT pg_last_wal_replay_lsn();"
# 0/1A000000

$ psql -h standby.db.bank.com -c "SELECT pg_last_wal_replay_lsn();"
# 0/1A000000  (same = no lag) ✅

# Step 3: Simulate primary failure
$ kubectl delete nodes -l site=mumbai --ignore-not-found
# node "node-mum-01" deleted
# node "node-mum-02" deleted
# node "node-mum-03" deleted

# Step 4: Monitor failover
$ watch kubectl get pods -l site=delhi
# [00:00] payment-del-01: Pending
# [00:05] payment-del-01: ContainerCreating
# [00:10] payment-del-01: Running
# [00:15] payment-del-02: Running
# [00:20] payment-del-03: Running

# Step 5: Verify DR cluster is serving traffic
$ curl -w "HTTP Code: %{http_code}\nTime: %{time_total}s\n" https://api.bank.com/health
# {"status":"healthy","region":"delhi","version":"2.5.0"}
# HTTP Code: 200
# Time: 0.045s

# Step 6: Verify data integrity
$ psql -h standby.db.bank.com -c "SELECT COUNT(*) FROM transactions WHERE created_at > NOW() - INTERVAL '1 hour';"
# 15,847 (matches primary count before failure) ✅

# Step 7: Failback to primary (after fixing)
# Rebuild Mumbai cluster
# Sync data from Delhi
# Switch DNS back to Mumbai
# Verify: primary healthy, replication restored
```

### E2E Example 2: DR Backup and Restore

**Context:** Test complete backup and restore procedure.

```bash
# Step 1: Full backup
$ pg_dump -Fc -h prod-db.bank.com -U backup_user banking > full_backup_$(date +%Y%m%d).dump
$ ls -lh full_backup_20260904.dump
# -rw-r--r-- 1 root root 45G Sep  4 10:00 full_backup_20260904.dump

# Step 2: Upload to cross-region storage
$ aws s3 cp full_backup_20260904.dump s3://bank-backups-delhi/production/ --storage-class GLACIER
# upload: ./full_backup_20260904.dump to s3://bank-backups-delhi/production/full_backup_20260904.dump

# Step 3: Test restore on staging
$ pg_restore -h staging-db.bank.com -U admin -d banking_test full_backup_20260904.dump
# Step 1: 14,892 ms, 542 tables
# Step 2: 8,234 ms, 542 FKs
# Step 3: 2,156 ms, 1,247 indexes
# Step 4: 45,892 ms, 2.3M rows
# Step 5: 1,892 ms, 1,247 sequences
# Step 6: 987 ms, 89 views

# Step 4: Verify data integrity
$ psql -h staging-db.bank.com -d banking_test -c "
    SELECT 
        (SELECT COUNT(*) FROM transactions) as transactions,
        (SELECT COUNT(*) FROM accounts) as accounts,
        (SELECT SUM(balance) FROM accounts) as total_balance
"
#  transactions | accounts | total_balance
# --------------+----------+---------------
#     2,345,678 |   123,456 |  1,234,567,890.00

# Step 5: Compare with production
$ psql -h prod-db.bank.com -c "SELECT COUNT(*) FROM transactions;"
# 2,345,678 ✅ (matches staging restore)
```

### E2E Example 3: Multi-Region Deployment with DR

**Context:** Deploy payment service across 3 regions with automatic failover.

```yaml
# Multi-region Helm values
# values-mumbai.yaml (Primary)
region: mumbai
role: primary
replicas: 6
database:
  host: postgres-mumbai.db.bank.com
  role: primary
  replication:
    mode: synchronous
    standby: postgres-delhi.db.bank.com

# values-delhi.yaml (DR)
region: delhi
role: standby
replicas: 3
database:
  host: postgres-delhi.db.bank.com
  role: standby
  replication:
    mode: asynchronous
    primary: postgres-mumbai.db.bank.com

# values-singapore.yaml (International)
region: singapore
role: active
replicas: 3
database:
  host: postgres-sg.db.bank.com
  role: primary
  replication:
    mode: asynchronous
    primary: postgres-mumbai.db.bank.com
```

```bash
# Deploy to all regions
$ helm upgrade --install payment ./helm/payment-chart -f values-mumbai.yaml -n production
$ helm upgrade --install payment ./helm/payment-chart -f values-delhi.yaml -n production
$ helm upgrade --install payment ./helm/payment-chart -f values-singapore.yaml -n production

# Verify multi-region setup
$ kubectl get pods -l app=payment --all-namespaces
# NAMESPACE    NAME                          READY   NODE
# production   payment-mumbai-abc123         1/1     node-mum-01
# production   payment-mumbai-def456         1/1     node-mum-02
# production   payment-mumbai-ghi789         1/1     node-mum-03
# production   payment-delhi-jkl012          1/1     node-del-01
# production   payment-delhi-mno345          1/1     node-del-02
# production   payment-sg-pqr678             1/1     node-sg-01
# production   payment-sg-stu901             1/1     node-sg-02

# DNS health check (Route 53)
$ aws route53 list-health-checks
# Health Check ID: abc123
# Domain: api.bank.com
# Type: HTTPS
# Interval: 30 seconds
# Failure threshold: 3
# Regions: ap-south-1, ap-south-2

# Automatic failover scenario:
# 1. Mumbai region goes down
# 2. Route 53 health check fails (3 consecutive failures = 90 seconds)
# 3. DNS automatically routes to Delhi
# 4. Delhi promotes standby database to primary
# 5. Traffic continues with < 2 minute downtime
```

---

## 📋 Interview Questions

### Q1: What is the difference between RPO and RTO?
**Answer:** **RPO** (Recovery Point Objective) = max acceptable data loss. RPO=0 means zero data loss (synchronous replication). RPO=1 hour means up to 1 hour of data can be lost. **RTO** (Recovery Time Objective) = max acceptable downtime. RTO=15min means systems must be back within 15 minutes. Banks typically require RPO=0 (zero data loss) and RTO<15min (minimal downtime).

### Q2: How do you implement zero data loss in DR?
**Answer:** (1) **Synchronous replication** — every write confirmed by both primary and standby before acknowledging. (2) **Write-ahead logging (WAL)** — PostgreSQL WAL shipped in real-time. (3) **Quorum writes** — write to 2 of 3 nodes before success. Trade-off: synchronous replication adds latency (5-10ms). For banking, this trade-off is worth it — data loss is unacceptable.

### Q3: What is a DR test and how often should banks run them?
**Answer:** A DR test simulates a disaster and verifies failover works. Types: (1) **Tabletop** — team discusses DR procedures (quarterly). (2) **Component test** — failover one component (monthly). (3) **Full simulation** — fail entire region (annually). Banks should run: component tests monthly, full simulations annually. RBI mandates at least one full DR test per year.

### Q4: How do you handle DNS failover for banking applications?
**Answer:** (1) **Health checks** — Route 53/Cloudflare monitors primary region every 30 seconds. (2) **TTL** — set DNS TTL to 60 seconds (fast propagation). (3) **Failover routing** — primary → DR, automatic on health check failure. (4) **Geographic routing** — route users to nearest healthy region. (5) **Manual override** — Ops team can force failover via API. Total failover time: 90-180 seconds (3 health check failures × 30-60 second intervals).

### Q5: What is the 3-2-1 backup rule and how does it apply to banking?
**Answer:** 3 copies of data, on 2 different media types, with 1 offsite. Banks implement: (1) **3 copies** — primary database, local replica, offsite backup. (2) **2 media** — SSD (primary) + tape/object storage (backup). (3) **1 offsite** — different geographic region (Mumbai → Delhi). Additional: backup encryption, regular restore testing, immutable backups (WORM storage for compliance).

---

## 📚 Summary

| Concept | Key Takeaway |
|---------|-------------|
| RPO | Max data loss (banks need 0) |
| RTO | Max downtime (banks need <15min) |
| Active-Passive | Primary + standby, manual failover |
| Active-Active | Both regions active, automatic failover |
| 3-2-1 Rule | 3 copies, 2 media, 1 offsite |
| Banking Relevance | Zero data loss, regulatory compliance |

**Next:** [22-Compliance-as-Code.md](./22-Compliance-as-Code.md) — Learn automated compliance checking.
