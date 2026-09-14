# 88. Database Replication Lag Causing Stale Reads

## Scenario

Your platform has PostgreSQL primary with 3 read replicas in AWS RDS. The read replicas are 30 seconds behind the primary. The application reads from replicas for reporting, dashboard, and search features. Users report that after submitting an order, the order isn't visible in the dashboard for 30+ seconds. Support tickets are increasing. The application previously had replication lag under 1 second. Traffic increased 3x last month due to a product launch. You have a large `orders` table (500M rows) with frequent writes, and batch jobs that run every hour updating millions of rows.

## Interviewer Question

"Read replicas are 30 seconds behind the primary. Users see stale data after writes. How do you diagnose replication lag and implement a strategy to handle eventual consistency?"

## What I Should Think About

- What is normal replication lag vs problematic lag
- How PostgreSQL streaming replication works (WAL streaming vs async)
- How to measure lag (replay_lag, pg_stat_replication)
- Common causes: long-running queries on replicas, replica CPU saturation, index rebuilds, table bloat
- The application's read-after-write consistency requirements
- How to handle read-after-write consistency (session stickiness, routing writes-then-reads to primary)
- Eventual consistency tolerance per use case
- Query load on replicas (heavy reporting queries)
- WAL growth and archival issues
- Replica instance sizing
- Monitoring lag metrics properly

## Ideal Answer

**Phase 1: Confirm and measure lag**

The first thing is to confirm whether lag is genuine or an artifact of how it's measured. `pg_stat_replication` on the primary shows `write_lag`, `flush_lag`, `replay_lag`. The fastest signal: `SELECT now() - pg_last_xact_replay_timestamp()` on the replica.

**Phase 2: Find the cause**

Lag can come from:
1. WAL production rate exceeds replica replay rate (heavy writes, heavy UPDATEs producing large WAL)
2. Replica CPU/memory/IOPS saturated (replica is too small or overloaded with reporting queries)
3. Long-running queries on replica blocking WAL replay (replay is single-threaded per replica in some modes)
4. WAL archival lag (archiver falling behind)

**Phase 3: Fix and prevent**

Immediate: reduce load on replicas; route heavy reporting queries away; add a replica.
Long-term: tune autovacuum, rewrite batch updates to be less WAL-intensive, implement read-after-write consistency route.

## Architecture

```
  ┌───────────────────────────────┐
  │  APPLICATION                  │
  ├───────────────────────────────┤
  │  OrderService   writes ───────┼───▶ PRIMARY (read/write)
  │  DashboardService reads ──────┼───▶ REPLICA 1
  │  ReportingService reads ──────┼───▶ REPLICA 2
  │  SearchService reads  ────────┼───▶ REPLICA 3
  └───────────────────────────────┘

  ┌───────────────────────────────────────────────────┐
  │  PRIMARY (db.r6g.4xlarge)                         │
  │   Write-heavy: orders, order_items, payments      │
  │   WAL rate: 200MB/min                            │
  │   WAL segment: aws_wal_metadata                  │
  │   ┌─────────────────┐                            │
  │   │ streaming rep   │ ──▶ replica1.replay        │
  │   │ streaming rep   │ ──▶ replica2.replay        │
  │   │ streaming rep   │ ──▶ replica3.replay        │
  │   └─────────────────┘                            │
  └────────────────────┬──────────────────────────────┘
                       │ WAL (streaming)
     ┌─────────────────┼─────────────────┐
     ▼                 ▼                  ▼
  ┌─────────┐      ┌─────────┐       ┌─────────┐
  │ REPLICA1│      │ REPLICA2│       │ REPLICA3│
  │ r6g.xl  │      │ r6g.2xl │       │ r6g.xl  │
  │ CPU 90%⚠️│     │ CPU 40% │       │ CPU 30% │
  │ reporting│     │ dashboard│      │ search  │
  │ batch jobs│    │ (light) │       │ (light) │
  └─────────┘      └─────────┘       └─────────┘
       ▲ lag 30s!      ▲ lag 5s        ▲ lag 2s

  Diagram: All replicas stream the same WAL.
  Replica1 is overloaded with reporting queries
  → replay_lag is highest there.
```

## Investigation

**Step 1: Measure the lag from the replica side**
```sql
-- On each replica
SELECT now() - pg_last_xact_replay_timestamp() AS lag_seconds;

-- If NULL → no lag (replica fully caught up)
```

**Step 2: Check replication status on the primary**
```sql
SELECT client_addr, client_hostname, state, sync_state,
       sent_lsn, write_lsn, flush_lsn, replay_lsn,
       write_lag, flush_lag, replay_lag
FROM pg_stat_replication;
```

**Step 3: Check replica resource usage**
```bash
# CPU, memory, IOPS on reads replicas
# CloudWatch: ReplicaLag, WriteIOPS, ReadIOPS, CPUUtilization
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS --metric-name ReplicaLag \
  --dimensions Name=DBInstanceIdentifier,Value=my-replica \
  --start-time $(date -u -d '-1 hour' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 --statistics Maximum \
  --output table
```

**Step 4: Check for long-running queries on replicas**
```sql
-- On the lagging replica
SELECT pid, state, now() - query_start AS duration, query,
       wait_event_type, wait_event
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY query_start ASC
LIMIT 20;
```

**Step 5: Check WAL rate and page writes**
```bash
# CloudWatch metrics on primary
# - WriteThroughput
# - WriteIOPS
# - TransactionCommitRate
# - DatabaseConnections
```

## Commands

```bash
# Check replication lag for all replicas
psql -c "SELECT client_addr, replay_lag, write_lag, flush_lag
         FROM pg_stat_replication;"

# Check lag on a specific replica
psql -c "SELECT now() - pg_last_xact_replay_timestamp() AS replay_lag;"

# Check replica query activity
psql -c "SELECT client_addr, application_name, state, query,
         now() - query_start AS duration
         FROM pg_stat_activity ORDER BY query_start;"

# Check WAL segment size on primary
psql -c "SELECT pg_current_wal_insert_lsn(), pg_current_wal_lsn();"

# Check if vacuum is running on replica (autovacuum can hold replay)
psql -c "SELECT pid, datname, state, query FROM pg_stat_activity
         WHERE query LIKE '%VACUUM%' OR query LIKE '%autovacuum%';"

# CloudWatch alarm: replica lag too high
aws cloudwatch put-metric-alarm \
  --alarm-name replica-lag-high \
  --namespace AWS/RDS --metric-name ReplicaLag \
  --dimensions Name=DBInstanceIdentifier,Value=my-replica \
  --statistic Maximum --period 120 --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 --alarm-actions arn:aws:sns:...:sre
```

## Root Cause

1. **Replica 1 under-provisioned for reporting load** - reports run heavy GROUP BY aggregations on `orders` (500M rows) consuming all CPU, starving WAL replay threads
2. **Batch UPDATE patterns generate large WAL** - A batch job updates `orders.status` for millions of rows hourly, generating huge WAL volume; with a single-writer WAL replay model, backlog accumulates
3. **Autovacuum running on replicas** - autovacuum on the replica for reporting tables competes for CPU and can block WAL replayer
4. **WAL archival delays** - archiver writing WAL files to S3 blocked by slow network/IOPS
5. **No read-after-write consistency handling** - app always reads from replicas even for immediately-affected data

## Immediate Mitigation

```bash
# 1. Reduce load on lagging replica:
#    - Route dashboard/search queries to other healthy replicas
#    - Reduce reporting query concurrency (prioritize critical reports)
#    - Lower shared_buffers/effective_cache_size?
#    → Actually: DON'T lower; instead, upgrade or add replica

# 2. Cancel heavy reporting queries on the lagging replica
SELECT pg_cancel_backend(pid)
FROM pg_stat_activity
WHERE state = 'active' AND now() - query_start > interval '5 minutes';

# 3. If lag is critical, failover routing:
#    - Temporarily route reads for high-consistency features to PRIMARY
#    - e.g., order confirmation page → primary reads

# 4. Use replicas selectively:
#    - dashboard-service → read replica 2 (light load)
#    - reporting-service → read replica 3 (dedicated for reports)
#    - search-service → queued/offloaded after write
```

## Permanent Fix

1. **Right-size read replicas** for workload (larger instance, more IOPS)
2. **Add dedicated reporting replica** isolated from app reads
3. **Implement read-after-write consistency**:
   - Session stickiness: bind the user's reads to primary until writes settle or use a cookie/`consistent_read` flag
   - For critical flows (order confirm): read primary for N seconds after write
4. **Batch update rewrites**: break batch UPDATE into smaller batches; use `LIMIT`/indexed loop instead of single huge UPDATE
5. **Watch for WAL generation**: monitor `wal_bytes` per table; consider partitioning large tables
6. **Tune autovacuum**: schedule autovacuum on replicas outside peak reporting hours
7. **Consider logical replication vs streaming replication** if cross-schema filtering needed
8. **Alert-level monitoring on lag** (5 warning/30 critical) with page routing control

## Monitoring

```bash
# RDS metrics
# - ReplicaLag (per instance)
# - CPUUtilization per replica
# - WriteThroughput, ReadThroughput on primary
# - StorageBurstBalance (Aurora/GP2)
# - FreeableMemory

# Postgres queries to schedule:
# - Replica replay time every minute (cron job → metrics)
# - pg_stat_replication lag per replica

# Alerts:
# - ReplicaLag > 5s (warning)
# - ReplicaLag > 30s (critical → auto page)
# - Replica CPU > 80% sustained 10min (warning)
# - Primary WAL production rate > X MB/s (warning)
```

## Security

- Replicas must be in private subnets with same security groups as primary
- Encryption at rest for replicas (RDS default)
- Replication traffic should be encrypted (RDS handles via TLS)
- Separate DB users per service (least privilege), disallow write DDL on replicas
- Unused replication slots must be removed (they hold WAL and cause disk fill)

## Production Considerations

- **HA**: Multi-AZ offers synchronous replication for failover, but read replicas lag; differentiate the two
- **RTO/RPO**: async replicas give RPO risk; consider synchronous_commit settings for critical writes
- **Scalability**: add/drop replicas easily with RDS; right-size dedicated reporting replica
- **Cost**: replicas double storage cost; a dedicated reporting replica justified by report load
- **Compliance**: stale data may violate SLAs for financial reports - consider primary reads for compliance-critical queries
- **Operational**: document the read-routing policy clearly; maintain runbook for lag incidents

## Senior-Level Answer

"First, I'd measure actual `replay_lag` per replica with `pg_stat_replication` and `pg_last_xact_replay_timestamp()`. The two biggest causes are replica overloaded with reporting queries starving the WAL replayer, and batch UPDATEs generating huge WAL volume. Immediate action: kill long CPU-bound reporting queries and route dashboard/search reads to lighter replicas, or temporarone reads to primary for consistency-critical endpoints. Long-term: separate a dedicated reporting replica, rewrite large batch updates into bounded loops, and implement read-after-write consistency by controlling routes (session stickiness, per-endpoint consistency levels). I'd also set up lag-based alerts and capacity checks so a replica doesn't get added without load testing."

## Architect-Level Answer

"Replication lag forces us to distinguish consistency requirements per workflow. I'd establish a consistency SLA matrix: order confirmation → strong consistency (primary reads), catalog → eventual consistency. Implement read-after-write routing via a proxy layer (e.g., write timestamp tracking + replica lag query) so only genuinely sensitive reads hit primary. Architecturally, batch jobs should be moved to dedicated read replicas or rewritten to be WAL-friendly (smaller, indexed writes). A separate analytics warehouse (or event-driven materialized views) reduces replica pressure. We should also add capacity planning for replicas based on peak WAL rate and reporting workload, and enforce that adding a replica includes a load test. If lag persists, consider logical replication or a stronger replay strategy, but the sustainable answer is workload isolation and consistency-tiered routing."

## Follow-Up Questions

1. "What is the difference between `write_lag`, `flush_lag`, and `replay_lag` and which matters most for users?"
2. "Explain how `synchronous_commit` interacts with read replicas and lag."
3. "How would you handle read-after-write consistency without sending every read to the primary?"
4. "What's the impact of `wal_level`, `max_wal_senders`, and `max_replication_slots` on lag?"
5. "If a heavy batch UPDATE runs hourly and the replica is behind, how do you schedule or split it to keep lag under 5 seconds?"