# 87. Database Connection Pool Exhausted

## Scenario

At 2:00 AM, your monitoring system alerts that the application is throwing `HikariPool-1 - Connection is not available, request timed out after 30000ms` errors. The errors started gradually 20 minutes ago and are now affecting 30% of requests. All 20 database connections in the pool are in use. Some connections are held for minutes instead of the expected milliseconds. The application is a Java Spring Boot service connecting to PostgreSQL via HikariCP. There are 10 application pods, each with a 20-connection pool. The PgBouncer in front of PostgreSQL shows 200 active server connections. Investigation shows one service endpoint (`/api/v1/reports/export`) is holding connections for an average of 90 seconds.

## Interviewer Question

"The application is throwing 'connection pool exhausted' errors. All connections are in use and some are held for minutes. How do you diagnose connection leaks, optimize the pool, and fix the immediate issue?"

## What I Should Think About

- Connection pool exhaustion symptoms vs. root causes
- Connection leaks (not closing connections properly)
- Long-running transactions holding connections
- Slow queries blocking pool connections
- Pool configuration tuning (maximumPoolSize, connectionTimeout)
- How to find which queries/threads hold connections longest
- Application-level debugging (thread dumps)
- Database-side connection inspection
- Distinguishing between a leak, a bottleneck, and a slow query problem
- Impact of pool exhaustion on other endpoints (cascading failure)
- Kotlin/Java/JPA transaction management

## Ideal Answer

**Phase 1: Triage**

The immediate question: is this a connection leak, or are queries just too slow? If connections are held for milliseconds normally and now held for minutes, either:
1. A slow query is blocking pool connections
2. A connection leak isn't returning connections
3. A transaction isn't committing/rolling back

**Phase 2: Diagnosis**

1. Check PgBouncer stats for connection ages and states
2. Inspect active PostgreSQL sessions for long-running queries
3. Look for `idle in transaction` sessions - these are suspected leaks
4. Get thread dumps from application nodes to see where threads are stuck
5. Check the specific `/api/v1/reports/export` endpoint code path

**Phase 3: Fix**

1. Immediate: kill long-running queries and idle-in-transaction sessions
2. Terminate the misbehaving endpoint or add a connection timeout
3. Tune HikariCP: `maximumPoolSize`, `connectionTimeout`, `leakDetectionThreshold`, `validationTimeout`
4. Fix the code: proper transaction scoping, connection cleanup

## Ideal Answer

**Finding the leak:**

In PostgreSQL, `pg_stat_activity` shows `state = 'idle in transaction'` - this is the classic signature of a connection leak where a transaction was opened but never committed or rolled back. The query appears as whatever query was last executed. In HikariCP, `leakDetectionThreshold` triggers a stack trace when connections exceed the threshold.

## Architecture

```
  NORMAL FLOW (connections held for ms):
  ┌─────────┐    ┌────────────┐    ┌───────────┐
  │ Request │───▶│ Hikari Pool│───▶│ PostgreSQL│
  │         │◀───│ 20 conns   │◀───│           │
  └─────────┘    │ obtained: 2ms│   └───────────┘
                 │ returned: 2ms│
                 │ held: 4ms   │
                 └────────────┘

  EXHAUSTED FLOW (connections held for minutes):
  ┌─────────┐    ┌────────────┐    ┌───────────┐
  │ Request │───▶│ Hikari Pool│───▶│ PostgreSQL│
  │ #15     │    │ 20/20 USED │    │ idle in   │
  │ (wait    │    │ ┌────────┐ │    │ transaction│
  │  30s then│    │ │conn 1  │ │    │ (leaked)  │
  │  timeout)│    │ │conn 2  │ │───▶│ idle in   │
  │ ERROR    │    │ │conn 3  │ │    │ transaction│
  └─────────┘    │ │...     │ │───▶│ idle in   │
                 │ │conn 20 │ │    │ transaction│
                 │ └────────┘ │    │ (leaked)  │
                 └────────────┘    └───────────┘

  Connection lifecycle leak:
  BEGIN (transaction) ──▶ Query ──▶ ...work... ──▶ (never COMMIT/ROLLBACK)
                                   └──▶ connection never returned to pool

  How to spot leaked connections:
  psql -c "SELECT pid, state, xact_start, now() - xact_start AS xact_age,
           now() - query_start AS query_age, query
           FROM pg_stat_activity
           WHERE state = 'idle in transaction'
           ORDER BY xact_start ASC;"
```

## Investigation

**Step 1: Check PgBouncer pool status**
```bash
psql -h localhost -p 6432 -U pgbouncer pgbouncer -c "SHOW POOLS;"
psql -h localhost -p 6432 -U pgbouncer pgbouncer -c "SHOW STATS;"

# Look for:
# - server_active or server_idle values
# - long-running transactions
```

**Step 2: Check PostgreSQL for idle-in-transaction sessions**
```sql
-- THE KEY QUERY: leak detection
SELECT pid, usename, client_addr, state, xact_start,
       now() - xact_start AS transaction_age,
       query
FROM pg_stat_activity
WHERE state IN ('idle in transaction', 'idle in transaction (aborted)')
ORDER BY xact_start ASC;

-- If you see sessions idle-in-transaction for minutes → LEAK
-- If you see sessions state='active' with query_running for minutes → SLOW QUERY
```

**Step 3: Check active queries blocking the pool**
```sql
SELECT pid, usename, application_name, client_addr,
       state, wait_event_type, wait_event,
       now() - query_start AS query_duration,
       left(query, 200) AS query_preview
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY query_start ASC
LIMIT 30;
```

**Step 4: Get application traces**
```bash
# Get thread dump from the stuck service (jstack)
jstack <pid> > threaddump.txt
# Look for:
# - Threads waiting on HikariPool.getConnection
# - SQL execution longer than expected
# - Uncleared resources (ResultSet/Statement not closed)

# HikariCP leak detection (enable then observe logs):
#   leakDetectionThreshold: 60000
# After enabling, Hikari logs stack traces of connections acquired but not released
```

**Step 5: Review application code**
```java
// BAD PATTERN - connection leak
public List<Report> export() {
    Connection conn = dataSource.getConnection();  // acquires
    Statement stmt = conn.prepareStatement("...");
    ResultSet rs = stmt.executeQuery();
    while (rs.next()) { /* build report */ }
    return reportList;
    // MISSING: conn.close(), stmt.close(), rs.close()
    // With Spring @Transactional? Check if transaction commits
}

// With Spring:
@Transactional // BAD if method throws and never rolls back properly
public Report export() { ... }
```

## Commands

```bash
# Terminate idle-in-transaction sessions older than 5 minutes
psql -c "SELECT pg_terminate_backend(pid)
         FROM pg_stat_activity
         WHERE state = 'idle in transaction'
         AND (now() - xact_start) > interval '5 minutes';"

# Kill a specific stuck connection
psql -c "SELECT pg_terminate_backend(12345);"

# HikariCP configuration - spring application.yml
# dataSource:
#   hikari:
#     maximumPoolSize: 20          # connection pool size
#     minimumIdle: 5
#     connectionTimeout: 30000     # wait for a connection
#     idleTimeout: 600000
#     maxLifetime: 1800000
#     leakDetectionThreshold: 60000 # logs stack trace for leaked conns
#     validationTimeout: 5000

# Check connection usage with Hikari metrics
# Add micrometer + prometheus:
#   hikaricp_connections_active
#   hikaricp_connections_pending
#   hikaricp_connections_timeout_total
#   hikaricp_connections_creation

# Check pg_stat_statements for total time
psql -c "SELECT query, calls, ROUND(mean_exec_time::numeric, 2) AS avg_ms,
         ROUND(total_exec_time::numeric, 2) AS total_ms
         FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;"
```

## Root Cause

1. **Connection leak in report export** - The `/api/v1/reports/export` endpoint acquires a connection at method level and never releases it when an exception occurs mid-report
2. **Transaction not closed on error** - The transaction started but wasn't committed or rolled back on exception paths
3. **Spring `@Transactional` misuse** - The export method is `@Transactional` but calls external systems (slow REST calls) INSIDE the transaction scope
4. **Cascading failure** - Once pool was exhausted, every request waited 30 seconds, amplifying the problem
5. **Under-provisioned pool** - 20 connections per pod for a 10-pod deployment = 200 total, but heavy report queries need more

## Immediate Mitigation

```bash
# 1. Kill leaked sessions
psql -c "SELECT pg_terminate_backend(pid)
         FROM pg_stat_activity
         WHERE state = 'idle in transaction'
         AND (now() - xact_start) > interval '3 minutes';"

# 2. Disable/stop the report export endpoint or add a circuit breaker
kubectl scale deployment/report-service --replicas=0

# 3. Increase pool temporarily if the volume is legitimate
# spring.datasource.hikari.maximumPoolSize: 40

# 4. Add statement_timeout to avoid runaway queries
ALTER SYSTEM SET statement_timeout = '30s';
SELECT pg_reload_conf();

# 5. Restart services to reclaim leaked connections
kubectl rollout restart deployment/api-service
```

## Permanent Fix

1. **Fix the code** - Use try-with-resources, ensure close() in finally blocks, or better: let Spring manage transactions solely via `@Transactional` service methods, never manual connections
2. **Never do I/O inside transactions** - Move external REST calls, file I/O, or long computations OUTSIDE the transactional scope
3. **Enable `leakDetectionThreshold`** in all environments (logged exceptions when leak suspected)
4. **Add connection metrics** to Prometheus/Grafana: active connections, pending, timeouts, acquitr
5. **Add statement_timeout** and `transaction_idle_timeout` in PostgreSQL config
6. **Apply circuit breakers** on db failure - fail fast on pool exhaustion using Resilience4j
7. **Implement connection validation** (`connectionTestQuery: SELECT 1`)

## Monitoring

```bash
# Hikari metrics (Micrometer):
# - hikaricp_connections_active
# - hikaricp_connections_idle
# - hikaricp_connections_pending (waiting to acquire)
# - hikaricp_connections_timeout_total (pool exhaustion rate)
# - hikaricp_connections_creation_rate
# - hikaricp_connections_release_rate

# PostgreSQL metrics:
# - pg_stat_activity state distribution (active / idle / idle in transaction)
# - pg_stat_database_xact_commit / rollback
# - pg_stat_user_tables tuples updated/inserted
# - lock wait time

# Alerts:
# - hikaricp_connections_pending > 0 for 2 minutes → WARNING
# - hikaricp_connections_timeout_total increasing → CRITICAL
# - pg_stat_activity state = 'idle in transaction' count > 5 → WARNING
# - Pool utilization (active/maximum) > 80% → WARNING
```

## Security

- Ensure DB user has only necessary grants (limit what leaked connections can do)
- Use `pg_terminate_backend` only with proper authorization (audit who terminates)
- Connection pool credentials should use least-privilege; separate read replica user from write
- Avoid logging query parameters (may contain PII)
- Connections to DB should be encrypted (SSL/TLS)

## Production Considerations

- **HA**: Pool exhaustion can cascade - a problem with one DB affects all pods. Use connection pooling at the DB layer (PgBouncer) to protect the DB from app pool churn
- **Scalability**: Right-size pools: too large → DB overloading; too small → starvation. A common heuristic: pool size = cores * 2 + 1
- **Cost**: Over-provisioning pools across many pods wastes connections and DB memory
- **Compliance**: Transaction audits matter - leaked transactions can leave inconsistent data
- **Operational**: Runbook for connection exhaustion should have a decision tree - if connections are all active → investigate slow queries; if idle-in-transaction → investigate leaks
- **RTO**: Fastest recovery is to kill idle-in-transaction sessions and restart pods; ensure this is documented

## Senior-Level Answer

"I'd immediately check `pg_stat_activity` for `idle in transaction` sessions with long `xact_start` - that's the signature of a connection leak. Terminate those sessions to restore pool capacity, then enable Hikari's `leakDetectionThreshold` and thread dumps to identify exactly which code path leaks connections. The report export endpoint is the culprit: it's doing external calls inside a `@Transactional` scope, holding the connection for the whole duration. The permanent fix is to move external I/O outside transactions, use proper connection handling, and set up monitoring on `hikaricp_connections_pending` and timeout counters. I'd also add statement and transaction idle timeouts at the PostgreSQL level as a safety net."

## Architect-Level Answer

"This scenario points to systemic issues beyond a single endpoint. I'd establish connection governance standards: no manual connection acquisition in application code (framework-managed only), mandatory `leakDetectionThreshold` in all environments, and a contract that transaction scopes must complete in milliseconds. From an architecture perspective, we should decouple heavy workloads (`/api/v1/reports/export`) from the transaction path - route them to read replicas with separate pools, or better, offload report generation to an asynchronous job queue (Kafka/RabbitMQ) so the reporting work never occupies transactional connections. We should also set up a connection budget model that allocates pool capacity per service per environment. The monitoring strategy should include pool metrics trend alerts that predict exhaustion before it happens."

## Follow-Up Questions

1. "What's the difference between `connectionTimeout`, `validationTimeout`, and `idleTimeout` in HikariCP, and how would you set each?"
2. "You find a connection leak in a shared service used by 5 teams. How do you enforce a fix when ownership is unclear?"
3. "How would you handle connection exhaustion if the pool is sized correctly but queries are genuinely slow?"
4. "Explain why increasing `maximumPoolSize` past a threshold can actually worsen database performance."
5. "How do you use `pg_stat_activity` to distinguish between a leak, a slow query, and a lock contention scenario?"