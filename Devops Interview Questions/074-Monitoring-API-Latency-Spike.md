# 74. API Latency Spike - Investigation Using Observability

## Scenario

At 2:15 PM, the API response time for your e-commerce platform spiked from a normal 100ms average to 5 seconds. The platform runs 20 microservices on Kubernetes. You have Prometheus (metrics), Grafana (dashboards), Loki (logs), and Jaeger (traces) available. Users are experiencing timeouts on product search and checkout. The incident is declared P2. You need to walk through the investigation from initial detection to root cause using all three pillars of observability.

## Interviewer Question

"API latency jumped from 100ms to 5 seconds. Walk me through your investigation using metrics, logs, and traces. How do you identify which of 20 services is the bottleneck?"

## What I Should Think About

- Three pillars: metrics, logs, traces — when to use each
- Latency analysis: RED metrics (Rate, Errors, Duration)
- Service dependency mapping
- Database query performance
- Network latency between services
- Resource saturation (CPU, memory, connections)
- Recent deployments or changes
- Traffic patterns (sudden increase?)

## Ideal Answer

**Phase 1: Initial Triage (Metrics - 2 minutes)**
- Check overall API latency in Grafana — confirm the spike timeline
- Identify which API endpoints are affected (product-search, checkout)
- Check error rates — are we seeing 5xx errors or just slow responses?
- Check request rate — is there a traffic spike?

**Phase 2: Service-Level Analysis (Metrics - 5 minutes)**
- Use RED metrics per service: rate, errors, duration
- Check each service's p50, p95, p99 latency
- Identify which service shows the latency increase first
- Check resource metrics: CPU, memory, network, disk I/O

**Phase 3: Deep Dive (Traces - 5 minutes)**
- Use Jaeger to trace a slow request through the service chain
- Find the span where latency is added
- Check for downstream dependency issues (database, cache, external API)

**Phase 4: Log Correlation (Logs - 3 minutes)**
- Query Loki for error logs in the identified service
- Correlate timestamps with trace data
- Look for slow query logs, timeout errors, connection pool exhaustion

**Phase 5: Root Cause Identification**
- Database query taking 4 seconds (full table scan due to missing index)
- Confirm with database metrics (connection pool, query latency)

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  REQUEST FLOW                            │
│                                                          │
│  User → API Gateway → Product Service → DB              │
│                         (2ms)         (4800ms!)         │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │           THREE PILLARS INVESTIGATION            │    │
│  │                                                   │    │
│  │  METRICS          LOGS           TRACES           │    │
│  │  ┌──────────┐    ┌──────────┐   ┌──────────┐    │    │
│  │  │ Grafana   │    │ Loki     │   │ Jaeger   │    │    │
│  │  │ Dashboard │    │ Query    │   │ Trace    │    │    │
│  │  │ - Latency │    │ - Errors │   │ - Spans  │    │    │
│  │  │ - Rate    │    │ - Slow   │   │ - Timing │    │    │
│  │  │ - Errors  │    │   queries│   │ - Deps   │    │    │
│  │  └─────┬─────┘    └────┬─────┘   └────┬─────┘    │    │
│  │        └───────────────┼──────────────┘           │    │
│  │                        ▼                           │    │
│  │              ┌──────────────────┐                  │    │
│  │              │   ROOT CAUSE     │                  │    │
│  │              │   DB Query: 4.8s │                  │    │
│  │              │   Missing Index  │                  │    │
│  │              └──────────────────┘                  │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

## Investigation

1. **Check overall system metrics:**
   ```promql
   # API Gateway latency (95th percentile)
   histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))
   
   # Request rate
   sum(rate(http_requests_total[5m])) by (service)
   
   # Error rate
   sum(rate(http_requests_total{status=~"5.."}[5m])) by (service) / sum(rate(http_requests_total[5m])) by (service)
   ```

2. **Check service-level latency:**
   ```promql
   # Product service latency
   histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{service="product-service"}[5m])) by (le, endpoint))
   
   # Compare with baseline
   histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{service="product-service"}[1h])) by (le, endpoint))
   ```

3. **Check database metrics:**
   ```promql
   # PostgreSQL query latency
   pg_stat_activity_query_duration_seconds{state="active"}
   
   # Connection pool usage
   pg_stat_activity_count{state="active"} / pg_settings_max_connections
   ```

4. **Query Loki for errors:**
   ```logql
   {service="product-service"} | json | level="error" | line_format "{{.timestamp}} {{.message}}"
   
   # Slow queries
   {job="postgresql"} | logfmt | duration > 1000ms | line_format "{{.query}} {{.duration}}"
   ```

5. **Check traces in Jaeger:**
   ```
   Service: product-service
   Operation: search
   Duration: > 1000ms
   Tags: http.status_code=200
   ```

## Commands

```bash
# 1. Check pod resource usage
kubectl top pods -n production | grep product-service

# 2. Check for recent deployments
kubectl rollout history deployment/product-service -n production

# 3. Check database connections
kubectl exec -it postgres-0 -n production -- psql -U admin -c "SELECT count(*) FROM pg_stat_activity;"

# 4. Check slow queries
kubectl exec -it postgres-0 -n production -- psql -U admin -c "SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state FROM pg_stat_activity WHERE state = 'active' AND now() - pg_stat_activity.query_start > interval '1 second';"

# 5. Check Redis cache hit rate
kubectl exec -it redis-0 -n production -- redis-cli INFO stats | grep keyspace

# 6. Create database index for slow query
kubectl exec -it postgres-0 -n production -- psql -U admin -d products -c "CREATE INDEX CONCURRENTLY idx_products_search ON products USING gin(to_tsvector('english', name || ' ' || description));"

# 7. Check Grafana dashboard
# Navigate to: Grafana → Dashboards → Microservices → API Latency
```

## Root Cause

| Root Cause | Evidence |
|---|---|
| Missing database index on products table | `EXPLAIN ANALYZE` shows sequential scan, 4.8s query time |
| Connection pool exhaustion | PgBouncer showing 100% pool utilization |
| Cascading timeout | Product service timeouts causing gateway retries |
| No query timeout configured | Slow queries not being terminated |

## Immediate Mitigation

1. **Create the missing index** (takes 30 seconds with `CONCURRENTLY`)
2. **Restart product-service pods** to clear connection pool
3. **Scale up product-service** replicas (2 → 4) to handle backlog
4. **Add query timeout** — `statement_timeout = '5s'` in PostgreSQL config

## Permanent Fix

1. Add database query performance monitoring (pg_stat_statements)
2. Implement connection pooling with PgBouncer
3. Add query timeout and slow query logging
4. Implement circuit breaker for database connections
5. Add database index review to deployment checklist
6. Set up automated index recommendations (pganalyze)

## Monitoring

- **p95/p99 latency alerts** per service endpoint
- **Database query latency** — alert if > 500ms
- **Connection pool utilization** — alert if > 80%
- **Cache hit rate** — alert if < 90%
- **Slow query log** — alert on queries > 1s
- **Grafana dashboard** — real-time latency visualization

## Security

- Database queries should use parameterized statements (prevent SQL injection)
- Connection pool credentials should be in Secrets Manager
- Slow query logs may contain sensitive data — implement log masking
- Database access should be restricted to application service accounts only

## Production Considerations

- **20 microservices** — need service mesh for distributed tracing
- Index creation should be done during low-traffic periods
- Consider using online schema change tools (gh-ost, pt-online-schema-change)
- Connection pool sizing: 2-4x CPU cores per database
- Implement database read replicas for read-heavy workloads

## Senior-Level Answer

"I'd follow a structured observability-driven investigation: (1) Start with metrics — check RED metrics (Rate, Errors, Duration) across all services to identify the affected service, (2) Use traces in Jaeger to follow a slow request through the service chain and identify the specific span where latency is added, (3) Correlate with logs in Loki to find error messages or slow query logs, (4) Cross-reference with database metrics to confirm the root cause. The key is using all three pillars together — metrics tell you WHAT, traces tell you WHERE, logs tell you WHY."

## Architect-Level Answer

"At the architectural level, this incident reveals gaps in our observability maturity. I'd implement: (1) SLOs for each service with latency budgets, (2) Automated root cause analysis using trace data and service dependency graphs, (3) Proactive alerting on latency anomalies using ML-based detection (Prometheus + Thanos), (4) Database observability as a first-class concern — query performance dashboards, connection pool monitoring, index health tracking, (5) Service mesh (Istio/Linkerd) for automatic distributed tracing and traffic management."

## Follow-Up Questions

1. "How do you correlate metrics, logs, and traces when they use different identifiers?"
2. "What's the difference between sampling strategies in distributed tracing (head-based vs. tail-based)?"
3. "How do you handle observability in a serverless architecture where there are no pods to monitor?"
4. "How would you implement automated anomaly detection for API latency?"
5. "What's your approach to observability as code — defining dashboards, alerts, and SLOs in version control?"
