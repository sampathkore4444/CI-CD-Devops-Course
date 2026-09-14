# 85. Database CPU at 95% During Traffic Spike

## Scenario

Your e-commerce platform launched a Black Friday marketing campaign that drove 10x normal traffic. Within 30 minutes, the PostgreSQL primary database CPU hit 95% and stayed there. Application response times jumped from 200ms to 12 seconds. Users are seeing "504 Gateway Timeout" errors. The application uses PostgreSQL 15 with PgBouncer for connection pooling (max 200 connections). There are 3 read replicas, but they're also showing 80%+ CPU. The database has 128GB RAM and 32 CPU cores. Slow query log shows multiple queries running 10+ seconds. You notice disk IOPS are also elevated. The connection pool shows active connections spiking from 50 to 198.

## Interviewer Question

"You receive an alert that database CPU is at 95%. Traffic has increased 10x due to a marketing campaign. Queries are taking 10+ seconds. Walk me through your complete investigation and resolution process. How do you diagnose slow queries, connection issues, and resolve the database bottleneck both immediately and long-term?"

## What I Should Think About

- How to quickly identify what's consuming CPU (queries, connections, background processes)
- Connection pool exhaustion vs. slow query analysis
- PgBouncer stats and connection state breakdown
- PostgreSQL `pg_stat_activity` for active query analysis
- Query plan analysis with `EXPLAIN ANALYZE`
- Index effectiveness and missing indexes
- Read replica load distribution
- Connection pool tuning under load
- Whether to shed load vs. optimize
- Horizontal vs. vertical scaling options
- Cache layer (Redis) effectiveness
- Long-term architecture: read/write splitting, query optimization, partitioning

## Ideal Answer

**Phase 1: Immediate Triage (First 5 minutes)**

First, confirm the scope and identify the fastest path to relief:

1. Check PgBouncer stats to see connection distribution
2. Query `pg_stat_activity` to identify the top resource-consuming queries
3. Check if any single query pattern dominates
4. Determine if the issue is query volume, query complexity, or missing indexes

**Phase 2: Diagnosis**

Use PostgreSQL internals to pinpoint the bottleneck:

1. `pg_stat_statements` - identify the top queries by total time and calls
2. `pg_stat_activity` - see what's currently running and for how long
3. `EXPLAIN ANALYZE` on the worst offenders to check for sequential scans, bad join strategies
4. Check `pg_stat_user_tables` for seq scan counts and dead tuples
5. Review PgBouncer `SHOW POOLS` and `SHOW STATS` for connection behavior

**Phase 3: Immediate Mitigation**

- Kill long-running queries that are blocking others
- Temporarily increase PgBouncer pool size if safe
- Enable query throttling or rate limiting at application level
- Route reporting queries to read replicas
- If needed, shed non-critical load (health checks, analytics)

**Phase 4: Resolution**

- Add missing indexes on the identified slow queries
- Rewrite queries that cause full table scans
- Scale read replicas to absorb read traffic
- Tune PgBouncer `query_wait_timeout` and pool sizing
- Consider adding a Redis cache for frequently accessed data

## Architecture

```
                    ┌──────────────────────────────────────────┐
                    │            Load Balancer (HAProxy)        │
                    └──────────┬───────────┬────────────────────┘
                               │           │
                    ┌──────────▼──┐  ┌─────▼──────────┐
                    │  App Node 1 │  │  App Node 2    │
                    │  (16 cores) │  │  (16 cores)    │
                    └──────┬──────┘  └──────┬─────────┘
                           │                │
                    ┌──────▼────────────────▼──────┐
                    │       PgBouncer (200 max)     │
                    │   transaction pooling mode     │
                    │   default_pool_size: 20        │
                    └──────┬────────────────┬──────┘
                           │                │
              ┌────────────▼──┐    ┌───────▼────────────┐
              │  PostgreSQL   │    │  Redis Cache        │
              │  PRIMARY      │    │  (session cache)    │
              │  32 cores     │    │  16GB               │
              │  128GB RAM    │    └────────────────────┘
              │  CPU: 95% ⚠️  │
              └──┬────┬───┬──┘
                 │    │   │
          ┌──────▼┐ ┌▼───▼┐ ┌▼──────┐
          │Read   │ │Read │ │Read   │
          │Replica│ │Rep.2│ │Rep.3  │
          │ 80%   │ │ 82% │ │ 79%   │
          └───────┘ └─────┘ └───────┘

  BEFORE (normal):
    CPU: 25%  |  Connections: 50/200  |  Queries: 500/s
    Slow queries: 2  |  Avg latency: 80ms

  DURING SPIKE:
    CPU: 95%  |  Connections: 198/200 |  Queries: 5000/s
    Slow queries: 340 |  Avg latency: 12000ms
```

## Investigation

**Step 1: Check PgBouncer connection pools**
```sql
-- Connect to PgBouncer admin
psql -h localhost -p 6432 -U pgbouncer pgbouncer

-- Check pool status
SHOW POOLS;
SHOW STATS;
SHOW CLIENTS;
SHOW SERVERS;

-- Look for:
-- - client_active near client_max (pool saturated)
-- - server_active near server_max (backend saturated)
-- - avg_wait_time increasing (queries waiting for connections)
```

**Step 2: Identify resource-hungry queries**
```sql
-- Connect to PostgreSQL primary
psql -h primary.db.internal -p 5432 -U app_user production_db

-- Check current active queries
SELECT pid, now() - pg_stat_activity.query_start AS duration,
       query, state, wait_event_type, wait_event
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC
LIMIT 20;

-- Check pg_stat_statements (if enabled)
SELECT query, calls, mean_exec_time, total_exec_time,
       rows, 100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0) AS hit_percent
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

**Step 3: Analyze slow queries with EXPLAIN**
```sql
-- For each slow query, analyze the plan
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE customer_id = 12345
AND status = 'pending' ORDER BY created_at DESC;

-- Look for:
-- - Seq Scan on large tables (missing index)
-- - Nested Loop with high row estimates
-- - Sort operations on disk (work_mem too low)
-- - Hash Join with bad memory estimates
```

**Step 4: Check table and index statistics**
```sql
-- Check for bloated tables
SELECT schemaname, relname, n_live_tup, n_dead_tup,
       last_vacuum, last_autovacuum, last_analyze
FROM pg_stat_user_tables
WHERE n_dead_tup > 100000
ORDER BY n_dead_tup DESC;

-- Check index usage
SELECT indexrelname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;

-- Check sequential scans on large tables
SELECT schemaname, relname, seq_scan, seq_tup_read, idx_scan
FROM pg_stat_user_tables
WHERE seq_scan > 10000
ORDER BY seq_tup_read DESC;
```

**Step 5: Check system-level resources**
```bash
# Check PostgreSQL processes CPU usage
ps aux --sort=-%cpu | head -20

# Check disk I/O
iostat -xz 1 5

# Check memory and swap
free -h
vmstat 1 5

# Check PostgreSQL locks
psql -c "SELECT locktype, relation::regclass, mode, granted,
         pid, usename FROM pg_locks WHERE NOT granted;"
```

## Commands

```bash
# Kill long-running queries (blocking others)
psql -c "SELECT pg_terminate_backend(pid)
         FROM pg_stat_activity
         WHERE state = 'active'
         AND query_start < now() - interval '30 seconds'
         AND pid != pg_backend_pid();"

# Check replication lag
psql -c "SELECT client_addr, state, sent_lsn, write_lsn,
         replay_lag FROM pg_stat_replication;"

# PgBouncer: pause traffic to primary (route reads to replicas)
psql -h localhost -p 6432 -U pgbouncer pgbouncer -c "PAUSE production_db;"

# Force PostgreSQL to use specific indexes
SET enable_seqscan = off;
EXPLAIN ANALYZE SELECT ...;

# Check current PgBouncer configuration
psql -h localhost -p 6432 -U pgbouncer pgbouncer -c "SHOW CONFIG;"

# Monitor real-time query execution
watch -n 1 "psql -c \"SELECT pid, state, query, wait_event
  FROM pg_stat_activity WHERE state='active' ORDER BY query_start;\""

# Add missing index (non-blocking)
CREATE INDEX CONCURRENTLY idx_orders_status_created
ON orders(status, created_at DESC);

# Check table bloat
SELECT pg_size_pretty(pg_total_relation_size('orders')) as total_size;
SELECT pg_size_pretty(pg_relation_size('orders')) as table_size;
SELECT pg_size_pretty(pg_indexes_size('orders')) as index_size;
```

## Root Cause

1. **Missing composite index** - Queries filtering on `status` + `created_at` were doing sequential scans on a 50M row table
2. **Connection pool exhaustion** - PgBouncer `default_pool_size` was 20, insufficient for 5000 QPS
3. **Inefficient query patterns** - N+1 queries from ORM generating excessive individual SELECTs
4. **Replica load not distributed** - All read replicas hit the same replica due to sticky sessions
5. **No query caching** - Hot data (top products) was being queried from DB every request

## Immediate Mitigation

```sql
-- 1. Kill the worst offenders
SELECT pg_terminate_backend(pid) FROM pg_stat_activity
WHERE state = 'active' AND query_start < now() - interval '15 seconds';

-- 2. Increase PgBouncer pool size temporarily
psql -h localhost -p 6432 -U pgbouncer pgbouncer -c "RELOAD;"

-- 3. Add connection limits to prevent runaway queries
ALTER ROLE app_user CONNECTION LIMIT 50;

-- 4. Temporarily disable non-critical features
-- Route /analytics/* to a separate endpoint returning cached data

-- 5. Enable statement_timeout for long queries
SET statement_timeout = '10s';
```

## Permanent Fix

1. **Add composite indexes** on all frequently queried column combinations
2. **Implement query caching** with Redis for hot data (product catalog, session data)
3. **Tune PgBouncer**: `default_pool_size=40`, `max_client_conn=500`, `query_wait_timeout=30`
4. **Implement query rate limiting** at application level per user/endpoint
5. **Optimize ORM queries** - eliminate N+1 patterns with eager loading
6. **Add read replica load balancing** using `pg_service.conf` with random rotation
7. **Implement connection circuit breaker** in application to fail fast when pool is saturated
8. **Database partitioning** for the largest tables by date range

## Monitoring

```bash
# PostgreSQL metrics to monitor (via pg_stat_statements + Prometheus exporter)
# - pg_stat_activity_count by state (active, idle, idle in transaction)
# - pg_stat_database_blks_hit (cache hit ratio)
# - pg_stat_database_xact_commit vs xact_rollback
# - pg_locks count by locktype and mode
# - pg_stat_replication_lag
# - pg_bouncer_pools_client_active / server_active

# Alert thresholds
# - CPU > 80% for 5 minutes (warning)
# - CPU > 90% for 2 minutes (critical)
# - Connection pool utilization > 80% (warning)
# - Slow query count > 50/minute (warning)
# - Replication lag > 10 seconds (warning)
# - Cache hit ratio < 95% (warning)
# - Query wait time > 100ms average (warning)

# Grafana dashboard panels
# 1. Connections over time (stacked: active, idle, waiting)
# 2. Query throughput (QPS) and latency (p50, p95, p99)
# 3. Cache hit ratio
# 4. Top 10 queries by total time
# 5. Lock wait time
# 6. Replication lag
```

## Security

- Ensure `pg_stat_statements` doesn't expose sensitive query data
- Connection pooling should use least-privilege database roles
- Monitoring endpoints should require authentication
- Audit logging for `pg_stat_activity` access
- Encrypt connections between PgBouncer and PostgreSQL (SSL)
- Rotate monitoring user credentials regularly

## Production Considerations

- **HA**: PgBouncer should run on each app node or as a sidecar to avoid single point of failure
- **Scalability**: Read replicas should be auto-scaled based on connection count and CPU
- **Cost**: Right-size the primary database instance; consider reserved instances for production
- **Compliance**: Query logs may contain PII - implement log redaction
- **Operational**: Runbook for database CPU alerts should include decision tree for mitigation
- **RTO**: Adding indexes CONCURRENTLY avoids locks but takes longer; plan for maintenance windows
- **Connection limits**: AWS RDS/Aurora have `max_connections` tied to instance size

## Senior-Level Answer

"First, I'd check PgBouncer stats and `pg_stat_activity` simultaneously to identify the bottleneck. If a few queries dominate, I'd `EXPLAIN ANALYZE` them to find missing indexes. I'd immediately kill queries running longer than 30 seconds that aren't critical. Then I'd ensure reads are routed to replicas and consider temporarily raising PgBouncer pool size. For the root cause, I'd add a composite index on the offending table using `CREATE INDEX CONCURRENTLY` to avoid locks. Long-term, I'd implement Redis caching for hot data, optimize ORM N+1 queries, add read replica load balancing, and set up `pg_stat_statements`-based monitoring with alerts on query latency percentiles."

## Architect-Level Answer

"Beyond the immediate fix, this incident reveals architectural gaps. We need a tiered caching strategy: CDN for static content, Redis for session and hot data, PgBouncer for connection management, and PostgreSQL for persistent storage. I'd implement a query firewall pattern where application queries go through a validation layer that checks for full table scans and N+1 patterns. For traffic spikes, we should implement circuit breakers and graceful degradation - non-critical features degrade under load. Long-term, consider read/write splitting with CQRS, table partitioning for large tables, and a capacity planning model that ties database sizing to traffic projections. I'd also establish a database performance review process where all queries must pass EXPLAIN ANALYZE benchmarks before deployment."

## Follow-Up Questions

1. "What would you do if `CREATE INDEX CONCURRENTLY` is taking too long on a 500GB table and users are still experiencing slow queries?"
2. "How would you handle this scenario if the database was Aurora PostgreSQL and you could leverage Aurora's storage layer?"
3. "Explain how you'd implement query caching with Redis while maintaining cache coherence when data is updated."
4. "What's the difference between PgBouncer's transaction mode and session mode, and which would you choose for this scenario?"
5. "How would you design a capacity planning model to predict when you need to scale the database before a known traffic spike like Black Friday?"
