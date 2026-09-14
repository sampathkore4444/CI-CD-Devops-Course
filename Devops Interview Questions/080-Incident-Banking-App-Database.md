# 80. Banking Application Slow - Database Connections at 100%

## Scenario

At 10:05 AM, the mobile banking application becomes extremely slow. At 10:10 AM, users begin receiving "transaction failed" errors. The mobile app serves 50,000 active users. CPU utilization across all Kubernetes nodes is normal (30%). Memory is at 60%. But database connections are at 100%. The application uses PostgreSQL with PgBouncer connection pooling. The connection pool size is 100. There are 15 microservices, each with its own connection pool of 20 connections. The database `max_connections` is set to 200.

## Interviewer Question

"Database connections are at 100%. The banking app is slow. Walk through the complete incident response: first 5 minutes, next 15 minutes, root cause identification, recovery, permanent fix, and postmortem."

## What I Should Think About

- Incident severity: P1 (banking application, customer impact)
- First 5 minutes: Triage and stabilize
- Next 15 minutes: Investigate root cause
- Root cause analysis: connection pool exhaustion
- Recovery: restore service, prevent recurrence
- Postmortem: blameless, actionable lessons
- Communication: status page, stakeholder updates
- Permanent fixes: connection pool tuning, query optimization

## Ideal Answer

**Timeline:**

**10:05 AM — Detection**
- Monitoring alerts fire: "Database connections at 100%"
- PagerDuty pages on-call engineer
- On-call engineer acknowledges within 2 minutes

**10:07 AM — Triage (First 5 Minutes)**
- Check Grafana dashboard — confirm connection pool exhaustion
- Check application logs — see "connection refused" errors
- Check database metrics — confirm active connections at max
- Declare P1 incident
- Assign Incident Commander

**10:10 AM — Stabilization**
- Kill long-running queries (> 30 seconds)
- Restart PgBouncer to clear stuck connections
- Scale up application pods (3 → 6) to distribute load
- Notify stakeholders: "Investigating slow performance"

**10:12 AM — Investigation (Next 15 Minutes)**
- Check `pg_stat_activity` for long-running queries
- Found: batch job running full table scan (45-second query)
- Batch job was triggered at 10:03 AM (scheduled every 6 hours)
- Connection pool exhausted because batch job holds 20 connections × 3 replicas = 60 connections
- Normal traffic uses 80 connections
- Total: 140 connections > max_connections (200) — but PgBouncer was limiting to 100

**10:15 AM — Root Cause Identified**
- Batch job has unoptimized query (missing index)
- Batch job holds connections for entire duration (no connection release)
- Connection pool too small for combined workload

**10:20 AM — Recovery**
- Kill batch job: `pg_terminate_backend(pid)`
- Create index on batch query table
- Scale PgBouncer pool from 100 → 150 connections
- Restart application pods
- Verify: connections drop to 60, latency returns to normal

**10:25 AM — Incident Resolved**
- Users can complete transactions
- Monitor for 30 minutes
- Update status page: "Service restored"

**10:30 AM — Communication**
- Send incident summary to stakeholders
- Schedule postmortem for next day

**Next Day — Postmortem**
- Blameless postmortem
- Action items: optimize batch query, increase pool size, add connection monitoring

## Architecture

```
┌─────────────────────────────────────────────────┐
│          INCIDENT TIMELINE                        │
│                                                   │
│  10:05   10:07   10:10   10:15   10:20   10:25  │
│    │       │       │       │       │       │     │
│    ▼       ▼       ▼       ▼       ▼       ▼     │
│  Alert   Triage  Declare  Root    Recover Resolved│
│  Fire    Check   P1      Cause   Fix DB   Monitor│
│          Dash    Scale   Found   Restart  30min  │
│          Board   Pods            Pods            │
│                                                   │
│  CONNECTION STATE:                                │
│  10:05: ████████████████████ 100% (100/100)     │
│  10:10: ████████████████████ 100% (100/100)     │
│  10:15: ████████████████████ 100% (100/100)     │
│  10:20: ████████░░░░░░░░░░░░  60% (60/100)      │
│  10:25: ██████░░░░░░░░░░░░░░  30% (30/100)      │
└─────────────────────────────────────────────────┘
```

## Investigation

1. **Check database connections:**
   ```sql
   -- Active connections
   SELECT count(*) FROM pg_stat_activity WHERE state = 'active';
   
   -- Connections by state
   SELECT state, count(*) FROM pg_stat_activity GROUP BY state;
   
   -- Long-running queries
   SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
   FROM pg_stat_activity
   WHERE state = 'active'
   AND now() - pg_stat_activity.query_start > interval '10 seconds'
   ORDER BY duration DESC;
   
   -- Connections by application
   SELECT application_name, count(*)
   FROM pg_stat_activity
   GROUP BY application_name;
   ```

2. **Check PgBouncer stats:**
   ```sql
   -- PgBouncer pool status
   SHOW POOLS;
   SHOW STATS;
   SHOW CLIENTS;
   SHOW SERVERS;
   
   -- Check for waiting clients
   SELECT count(*) FROM pg_stat_activity WHERE wait_event IS NOT NULL;
   ```

3. **Check application logs:**
   ```bash
   # Check for connection errors
   kubectl logs -n production -l app=banking-api --since=30m | grep -i "connection\|timeout\|refused"
   
   # Check for slow queries
   kubectl logs -n production -l app=postgresql --since=30m | grep -i "slow\|duration\|timeout"
   ```

4. **Identify the batch job:**
   ```bash
   # Check for scheduled jobs
   kubectl get cronjobs -n production
   
   # Check batch job logs
   kubectl logs -n production -l app=batch-job --since=1h | tail -50
   ```

## Commands

```bash
# 1. Terminate long-running queries
kubectl exec -it postgres-0 -n production -- psql -U admin -c "
  SELECT pg_terminate_backend(pid)
  FROM pg_stat_activity
  WHERE state = 'active'
  AND now() - query_start > interval '30 seconds'
  AND pid != pg_backend_pid();"

# 2. Restart PgBouncer
kubectl rollout restart deployment/pgbouncer -n production

# 3. Scale application pods
kubectl scale deployment/banking-api --replicas=6 -n production

# 4. Create missing index
kubectl exec -it postgres-0 -n production -- psql -U admin -d banking -c "
  CREATE INDEX CONCURRENTLY idx_transactions_batch
  ON transactions(created_at, status)
  WHERE status = 'pending';"

# 5. Increase PgBouncer pool size
kubectl patch configmap pgbouncer-config -n production --type merge -p '{
  "data": {
    "pgbouncer.ini": "[databases]\nbanking = host=postgres port=5432\n\n[pgbouncer]\nmax_client_conn = 200\ndefault_pool_size = 150"
  }
}'

# 6. Restart PgBouncer with new config
kubectl rollout restart deployment/pgbouncer -n production

# 7. Verify connections are back to normal
kubectl exec -it postgres-0 -n production -- psql -U admin -c "SELECT count(*) FROM pg_stat_activity;"
```

## Root Cause

| Root Cause | Evidence | Fix |
|---|---|---|
| Unoptimized batch query | 45-second full table scan | Add index, optimize query |
| Batch job holds connections too long | Connections held for entire duration | Implement connection release after each batch |
| Connection pool too small | Pool exhaustion under combined load | Increase pool size from 100 → 150 |
| No connection timeout | Queries running indefinitely | Set `statement_timeout = '30s'` |
| No batch job monitoring | Batch job impact not visible | Add batch job metrics and alerts |

## Immediate Mitigation

1. **Kill long-running queries** — `pg_terminate_backend(pid)`
2. **Restart PgBouncer** — clear stuck connections
3. **Scale application pods** — distribute connection load
4. **Monitor** for 30 minutes to confirm stability

## Permanent Fix

1. Optimize batch query with proper index
2. Implement connection release in batch job (connection per query, not per job)
3. Increase PgBouncer pool size to 150
4. Add `statement_timeout = '30s'` to PostgreSQL
5. Implement batch job scheduling with connection-aware logic
6. Add connection pool monitoring and alerts

## Monitoring

- **Connection pool utilization** — alert at 80%
- **Active queries** — alert on queries > 10 seconds
- **Batch job duration** — alert if > 5 minutes
- **Application connection errors** — alert on "connection refused"
- **PgBouncer stats** — clients waiting, pool utilization

## Security

- Batch job should use dedicated service account with limited permissions
- Connection credentials should be in Secrets Manager
- Database access should be logged (pgAudit)
- Implement connection rate limiting per service

## Production Considerations

- **50,000 active users** — connection pool must handle peak traffic
- Connection pool sizing: 2-4x CPU cores per database
- Consider read replicas for batch queries
- Implement connection circuit breaker in application
- Cost: PostgreSQL RDS instance may need upgrade for higher connection limits

## Senior-Level Answer

"I'd follow a structured incident response: (1) First 5 minutes — triage, declare P1, assign IC, (2) Next 15 minutes — investigate with database metrics and query analysis, (3) Root cause — batch job with unoptimized query exhausting connection pool, (4) Recovery — kill queries, restart PgBouncer, scale pods, (5) Permanent fix — optimize query, tune pool, add monitoring. The key insight is that connection pool exhaustion is usually a symptom of an underlying query performance issue."

## Architect-Level Answer

"At the architectural level, I'd establish: (1) Connection Pool Architecture — centralized connection management with PgBouncer, (2) Batch Job Standards — all batch jobs must release connections after each query, (3) Database Observability — real-time connection pool monitoring and alerts, (4) Incident Response Playbooks — pre-built runbooks for common issues like connection exhaustion, (5) Capacity Planning — model connection requirements based on traffic patterns."

## Follow-Up Questions

1. "How do you implement connection pooling with PgBouncer in a Kubernetes environment?"
2. "What's the difference between connection pooling and query caching?"
3. "How do you handle connection pool exhaustion in a microservices architecture?"
4. "How would you implement circuit breaker pattern for database connections?"
5. "What's your approach to database connection monitoring and alerting?"
