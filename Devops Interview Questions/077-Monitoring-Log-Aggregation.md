# 77. Log Aggregation and Analysis in Production

## Scenario

A critical production incident is occurring — the payment service is returning 500 errors. You need to find the root cause within 15 minutes. The system has 50+ Kubernetes containers, 5 PostgreSQL databases, and 3 load balancers (HAProxy). Logs are spread across Loki (Kubernetes pods), ELK (application logs), and CloudWatch (AWS services). You have Grafana for visualization. The on-call engineer says "I can see errors in the logs but can't correlate them across services." How do you efficiently search, correlate, and analyze logs across all these systems?

## Interviewer Question

"You have 15 minutes to find the root cause across 50+ containers, 5 databases, and 3 load balancers. Logs are in Loki and ELK. How do you efficiently search and correlate logs across all these systems?"

## What I Should Think About

- Log aggregation architecture (Loki vs. ELK vs. CloudWatch)
- Structured logging (JSON format) for efficient querying
- LogQL (Loki) and KQL (KQL) query syntax
- Correlation across services (trace ID, timestamp, request ID)
- Log-based alerts and anomaly detection
- Log retention and cost optimization
- Performance of log queries at scale

## Ideal Answer

**1. Structured Logging (Foundation)**
- All services output JSON logs with standardized fields
- Common fields: `timestamp`, `level`, `service`, `trace_id`, `span_id`, `message`
- Business fields: `user_id`, `order_id`, `payment_id`

**2. Correlation Strategy**
- Use trace ID as the correlation key across all services
- Add request ID to all log entries
- Use consistent timestamp format (ISO 8601 with timezone)

**3. Query Strategy (Loki)**
- Use LogQL for efficient label-based queries
- Filter by service, then by trace ID, then by message pattern
- Use `| json` parser for structured logs
- Use `| line_format` for readable output

**4. Query Strategy (ELK/Kibana)**
- Use KQL (Kibana Query Language) for quick filtering
- Use Lucene for complex queries
- Create saved searches for common patterns

**5. Investigation Workflow**
- Start broad: all errors in last 15 minutes
- Narrow by service: payment-service errors
- Correlate with trace ID: find related logs across services
- Deep dive: specific error message and stack trace

## Architecture

```
┌─────────────────────────────────────────────────┐
│           LOG AGGREGATION ARCHITECTURE            │
│                                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │ App Pods │  │ DB Logs  │  │ LB Logs  │      │
│  │ (JSON)   │  │ (JSON)   │  │ (JSON)   │      │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘      │
│       │              │              │             │
│       ▼              ▼              ▼             │
│  ┌──────────────────────────────────────────┐   │
│  │         LOG AGGREGATION LAYER            │   │
│  │                                           │   │
│  │  ┌──────────┐  ┌──────────┐  ┌────────┐ │   │
│  │  │ Promtail │  │ Filebeat │  │ Fluentd│ │   │
│  │  │ (K8s)    │  │ (DB)     │  │ (LB)   │ │   │
│  │  └────┬─────┘  └────┬─────┘  └───┬────┘ │   │
│  └───────┼──────────────┼────────────┼──────┘   │
│          ▼              ▼            ▼           │
│  ┌──────────────────────────────────────────┐   │
│  │         STORAGE & QUERY                  │   │
│  │  ┌──────────┐  ┌──────────┐  ┌────────┐ │   │
│  │  │ Loki     │  │ ELK      │  │CloudWatch│ │  │
│  │  │ (K8s)    │  │ (App)    │  │ (AWS)   │ │   │
│  │  └────┬─────┘  └────┬─────┘  └───┬────┘ │   │
│  └───────┼──────────────┼────────────┼──────┘   │
│          ▼              ▼            ▼           │
│  ┌──────────────────────────────────────────┐   │
│  │              GRAFANA                      │   │
│  │  ┌──────────┐  ┌──────────┐  ┌────────┐ │   │
│  │  │ Dashboards│  │ Explore  │  │ Alerts │ │   │
│  │  └──────────┘  └──────────┘  └────────┘ │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

## Investigation

1. **Start with broad error search:**
   ```logql
   # Loki: All errors across all services in last 15 minutes
   {namespace="production"} | json | level="error" | line_format "{{.timestamp}} {{.service}} {{.message}}"
   
   # ELK/Kibana KQL:
   level:error AND kubernetes.namespace:production AND @timestamp:[now-15m TO now]
   ```

2. **Narrow by service:**
   ```logql
   # Loki: Payment service errors
   {namespace="production", app="payment-service"} | json | level="error" | line_format "{{.timestamp}} {{.message}} {{.stack_trace}}"
   
   # Check for specific error pattern
   {namespace="production", app="payment-service"} |~ "payment.*failed|timeout|connection refused"
   ```

3. **Correlate with trace ID:**
   ```logql
   # Find trace ID from payment service error
   {namespace="production", app="payment-service"} | json | level="error" | line_format "{{.trace_id}}"
   
   # Use trace ID to find related logs across ALL services
   {namespace="production"} | json | trace_id="abc123def456" | line_format "{{.service}} {{.message}}"
   ```

4. **Check database logs:**
   ```logql
   # Loki: PostgreSQL slow queries
   {namespace="production", app="postgresql"} | logfmt | duration > 1000 | line_format "{{.query}} {{.duration}}"
   
   # Connection errors
   {namespace="production", app="postgresql"} |~ "connection.*refused|too many connections"
   ```

5. **Check load balancer logs:**
   ```logql
   # HAProxy errors
   {namespace="production", app="haproxy"} |~ "500|502|503|504" | line_format "{{.client_ip}} {{.request}} {{.status}}"
   ```

## Commands

```bash
# 1. Query Loki for payment service errors (LogQL)
logcli query '{namespace="production", app="payment-service"} | json | level="error"' --since=15m --limit=100

# 2. Query with aggregation
logcli query 'sum(count_over_time({namespace="production", app="payment-service"} | json | level="error" [1m])) by (service)' --since=15m

# 3. Query ELK via API
curl -s "http://elasticsearch:9200/logs-*/_search" -H "Content-Type: application/json" -d '{
  "query": {
    "bool": {
      "must": [
        {"match": {"level": "error"}},
        {"range": {"@timestamp": {"gte": "now-15m"}}},
        {"match": {"kubernetes.namespace": "production"}}
      ]
    }
  },
  "sort": [{"@timestamp": {"order": "desc"}}],
  "size": 50
}' | jq '.hits.hits[]._source | {timestamp: .["@timestamp"], service: .kubernetes.pod.name, message: .message}'

# 4. Search for specific trace ID
logcli query '{namespace="production"} | json | trace_id="abc123def456"' --since=1h --limit=200

# 5. Find slow database queries
logcli query '{namespace="production", app="postgresql"} | logfmt | duration > 1000000000' --since=1h --limit=50

# 6. Export logs for analysis
logcli query '{namespace="production", app="payment-service"} | json | level="error"' --since=15m --output=csv > payment-errors.csv
```

## Root Cause

| Root Cause | Investigation Step |
|---|---|
| Database connection pool exhaustion | Check PostgreSQL logs for "too many connections" |
| Payment gateway timeout | Check payment service logs for "timeout" errors |
| Memory leak in payment service | Check container OOM kills in Kubernetes events |
| HAProxy backend server down | Check HAProxy logs for 502/503 errors |
| Disk full on database | Check database logs for "disk full" errors |

## Immediate Mitigation

1. **Identify the specific error** — use trace ID to correlate across services
2. **Check database connections** — `kubectl exec postgres-0 -- psql -c "SELECT count(*) FROM pg_stat_activity;"`
3. **Check for OOM kills** — `kubectl describe pods -n production | grep -A5 "Last State"`
4. **Restart affected pods** if needed — `kubectl rollout restart deployment/payment-service -n production`

## Permanent Fix

1. Implement structured JSON logging across all services
2. Standardize log fields (timestamp, level, service, trace_id)
3. Configure Loki/ELK retention policies (7 days hot, 30 days warm, 90 days cold)
4. Create log-based alerts for error rate spikes
5. Implement log-based dashboards in Grafana

## Monitoring

- **Error rate by service** — alert if > 1% of requests
- **Log volume anomalies** — alert on sudden spikes/drops
- **Slow query detection** — alert on queries > 1s
- **Log ingestion lag** — alert if > 30 seconds behind
- **Storage utilization** — alert at 80% capacity

## Security

- Logs may contain sensitive data (PII, credentials)
- Implement log masking for sensitive fields
- Restrict log access to authorized personnel
- Use audit logs for compliance (SOC2, PCI-DSS)
- Implement log retention policies per compliance requirements

## Production Considerations

- **50+ containers** — need efficient log aggregation (Promtail + Loki)
- **5 databases** — need database-specific log collection
- **3 load balancers** — need HAProxy log parsing
- Cost: Loki storage ~$0.50/GB/month, ELK ~$2/GB/month
- Query performance: Loki uses labels (fast), ELK uses full-text (flexible)

## Senior-Level Answer

"I'd use a structured approach: (1) Start with broad error search across all services, (2) Narrow by service using labels/tags, (3) Correlate across services using trace ID, (4) Deep dive into specific error messages and stack traces. The key is structured logging with trace ID correlation — without that, you're doing grep across multiple systems."

## Architect-Level Answer

"At the architectural level, I'd establish: (1) Logging Standards — all services must output JSON logs with standard fields, (2) Centralized Log Management — single platform (Loki preferred for cost, ELK for flexibility), (3) Trace-Log Correlation — trace ID in all logs for cross-service debugging, (4) Log-Based Observability — alerts and dashboards from logs, not just metrics, (5) Cost Optimization — tiered storage (hot/warm/cold) and retention policies."

## Follow-Up Questions

1. "What's the difference between Loki and ELK? When would you choose one over the other?"
2. "How do you implement log-based alerts that don't create alert fatigue?"
3. "How do you handle log correlation across Kubernetes and external databases?"
4. "How would you implement log-based anomaly detection for proactive incident detection?"
5. "What's your approach to log retention and cost optimization at scale?"
