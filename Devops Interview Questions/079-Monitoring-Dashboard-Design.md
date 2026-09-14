# 79. Dashboard Design for Operations Team

## Scenario

You need to create Grafana dashboards for a microservices platform with 20 services running on Kubernetes. The operations team complains that current dashboards are "walls of graphs" — 50+ panels with no clear hierarchy. During the last incident, the on-call engineer spent 10 minutes scrolling through graphs before finding the relevant metric. Management wants "health at a glance" — a single view that immediately shows if the system is healthy or not. The platform includes API Gateway, 15 microservices, 3 databases (PostgreSQL, Redis, Elasticsearch), and message queues (Kafka).

## Interviewer Question

"How do you design Grafana dashboards that are actionable during incidents — showing system health at a glance and enabling rapid root cause analysis?"

## What I Should Think About

- Dashboard hierarchy: Executive → Service → Debug
- The "Golden Signals" (Google SRE): Latency, Traffic, Errors, Saturation
- RED metrics (Rate, Errors, Duration) per service
- USE metrics (Utilization, Saturation, Errors) for infrastructure
- Dashboard organization: overview → drill-down
- Color coding and threshold design
- Dashboard templating (variable dropdowns)
- Dashboard as code (Grafana provisioning)
- Anti-patterns: walls of graphs, too many panels, no context

## Ideal Answer

**1. Dashboard Hierarchy**
- **Level 1: Executive Overview** — System health at a glance (5 panels)
- **Level 2: Service Dashboard** — Per-service RED metrics (15 panels)
- **Level 3: Debug Dashboard** — Deep dive for investigation (30+ panels)

**2. Level 1: Executive Overview (5 panels)**
- Overall availability (big number, green/yellow/red)
- Request rate (traffic)
- Error rate (with threshold)
- p95 latency (with SLO line)
- Active incidents (count)

**3. Level 2: Service Dashboard (15 panels)**
- Service request rate, error rate, latency (3 panels)
- Service CPU, memory, network (3 panels)
- Database queries, connections, latency (3 panels)
- Cache hit rate, memory, evictions (3 panels)
- Dependency health (3 panels)

**4. Level 3: Debug Dashboard (30+ panels)**
- Detailed per-endpoint metrics
- Pod-level metrics
- Database query analysis
- Log volume and errors
- Trace latency distribution

**5. Design Principles**
- Top-left to bottom-right flow (most important first)
- Color coding: green = healthy, yellow = warning, red = critical
- Time range: default to last 1 hour
- Variables: service dropdown, environment dropdown
- Annotations: deployments, incidents, maintenance

## Architecture

```
┌─────────────────────────────────────────────────┐
│           DASHBOARD HIERARCHY                    │
│                                                   │
│  ┌──────────────────────────────────────────┐   │
│  │  LEVEL 1: EXECUTIVE OVERVIEW (5 panels)  │   │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐   │   │
│  │  │Avail │ │Rate  │ │Errors│ │Latency│   │   │
│  │  │99.99%│ │1.2k/s│ │0.01% │ │120ms │   │   │
│  │  │  🟢  │ │  🟢  │ │  🟢  │ │  🟢  │   │   │
│  │  └──────┘ └──────┘ └──────┘ └──────┘   │   │
│  │  ┌──────────────────────────────────┐   │   │
│  │  │ Active Incidents: 0              │   │   │
│  │  └──────────────────────────────────┘   │   │
│  └──────────────────────────────────────────┘   │
│                        │ click to drill down     │
│                        ▼                         │
│  ┌──────────────────────────────────────────┐   │
│  │  LEVEL 2: SERVICE DASHBOARD (15 panels)   │   │
│  │  ┌────────────────────────────────────┐  │   │
│  │  │ RED Metrics (Request, Error, Dur)  │  │   │
│  │  └────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────┐  │   │
│  │  │ USE Metrics (CPU, Mem, Network)    │  │   │
│  │  └────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────┐  │   │
│  │  │ Database & Cache Metrics           │  │   │
│  │  └────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────┘   │
│                        │ click to drill down     │
│                        ▼                         │
│  ┌──────────────────────────────────────────┐   │
│  │  LEVEL 3: DEBUG DASHBOARD (30+ panels)    │   │
│  │  ┌────────────────────────────────────┐  │   │
│  │  │ Per-endpoint metrics, pod-level    │  │   │
│  │  │ Database queries, logs, traces     │  │   │
│  │  └────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

## Investigation

1. **Design Level 1 overview dashboard:**
   ```json
   {
     "panels": [
       {
         "title": "System Availability",
         "type": "stat",
         "gridPos": {"h": 6, "w": 6, "x": 0, "y": 0},
         "targets": [{
           "expr": "sum(rate(http_requests_total{status!~\"5..\"}[5m])) / sum(rate(http_requests_total[5m])) * 100"
         }],
         "fieldConfig": {
           "defaults": {
             "thresholds": {
               "steps": [
                 {"value": 0, "color": "red"},
                 {"value": 99.9, "color": "yellow"},
                 {"value": 99.99, "color": "green"}
               ]
             },
             "unit": "percent"
           }
         }
       }
     ]
   }
   ```

2. **Design Level 2 service dashboard:**
   ```json
   {
     "panels": [
       {
         "title": "Request Rate",
         "type": "timeseries",
         "gridPos": {"h": 8, "w": 8, "x": 0, "y": 0},
         "targets": [{
           "expr": "sum(rate(http_requests_total{service=\"$service\"}[5m])) by (endpoint)"
         }]
       },
       {
         "title": "Error Rate",
         "type": "timeseries",
         "gridPos": {"h": 8, "w": 8, "x": 8, "y": 0},
         "targets": [{
           "expr": "sum(rate(http_requests_total{service=\"$service\",status=~\"5..\"}[5m])) / sum(rate(http_requests_total{service=\"$service\"}[5m])) * 100"
         }]
       },
       {
         "title": "p95 Latency",
         "type": "timeseries",
         "gridPos": {"h": 8, "w": 8, "x": 16, "y": 0},
         "targets": [{
           "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{service=\"$service\"}[5m])) by (le))"
         }]
       }
     ],
     "templating": {
       "list": [{
         "name": "service",
         "type": "query",
         "query": "label_values(http_requests_total, service)"
       }]
     }
   }
   ```

3. **Create dashboard provisioning config:**
   ```yaml
   # /etc/grafana/provisioning/dashboards/dashboard.yml
   apiVersion: 1
   providers:
     - name: 'default'
       orgId: 1
       folder: 'Microservices'
       type: file
       disableDeletion: false
       updateIntervalSeconds: 30
       options:
         path: /var/lib/grafana/dashboards
   ```

## Commands

```bash
# 1. Export dashboard as JSON for version control
curl -s "http://grafana:3000/api/dashboards/uid/system-overview" | jq '.dashboard' > dashboards/system-overview.json

# 2. Provision dashboards via Terraform
cat > grafana-dashboard.tf << 'EOF'
resource "grafana_dashboard" "system_overview" {
  config_json = file("dashboards/system-overview.json")
  folder      = grafana_folder.microservices.id
}
EOF

# 3. Create alerts from dashboard panels
# In Grafana UI: Panel → Alert → Create Alert
# Or via API:
curl -s -X POST "http://grafana:3000/api/v1/provisioning/alert-rules" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "High Error Rate",
    "condition": "A",
    "data": [{
      "model": {
        "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m])) * 100 > 1"
      }
    }]
  }'

# 4. Query dashboard statistics
curl -s "http://grafana:3000/api/search?query=system" | jq '.[] | {title: .title, uid: .uid, url: .url}'
```

## Root Cause

| Root Cause | Elimination |
|---|---|
| No dashboard hierarchy | Implement 3-level hierarchy (Executive → Service → Debug) |
| Too many panels per dashboard | Max 10-15 panels per dashboard |
| No clear health indicators | Use color-coded stat panels (green/yellow/red) |
| No templating | Add service/environment variable dropdowns |
| Dashboards not version controlled | Store dashboard JSON in Git |
| No drill-down capability | Link dashboards (click panel → related dashboard) |

## Immediate Mitigation

1. **Create Level 1 Executive Overview** — 5 panels showing system health
2. **Add color coding** to existing dashboards
3. **Add service dropdown** for quick filtering
4. **Add annotations** for deployments and incidents

## Permanent Fix

1. Implement 3-level dashboard hierarchy
2. Use dashboard-as-code (JSON in Git)
3. Provision dashboards via Terraform/Grafana provisioning
4. Regular dashboard review (monthly)
5. Train team on dashboard navigation

## Monitoring

- **Dashboard usage analytics** — which dashboards are used during incidents
- **Mean Time to Identify (MTTI)** — how fast can team find root cause
- **Dashboard freshness** — are dashboards up-to-date with current architecture
- **User feedback** — regular survey on dashboard usefulness

## Security

- Dashboard access should follow RBAC (read-only for most users)
- Sensitive metrics (revenue, user count) should be restricted
- Dashboard URLs should not expose sensitive query parameters
- Use Grafana folders and teams for access control

## Production Considerations

- **20 services** — need templated dashboards for consistency
- Dashboard load time — optimize queries for fast rendering
- Consider using Grafana panels with consistent color schemes
- Cost: Grafana Cloud $8/month for basic, $29/month for pro
- Consider using Grafana annotations for deployment tracking

## Senior-Level Answer

"I'd implement a 3-level dashboard hierarchy: (1) Executive Overview — 5 panels showing system health at a glance, (2) Service Dashboard — per-service RED metrics with templating, (3) Debug Dashboard — deep dive for investigation. The key principle is 'health at a glance' — within 5 seconds, the operator should know if the system is healthy or not."

## Architect-Level Answer

"At the architectural level, I'd establish: (1) Dashboard Standards — consistent layout, color coding, and naming conventions across all teams, (2) Dashboard-as-Code — all dashboards stored in Git and provisioned via CI/CD, (3) Dashboard Governance — monthly review to ensure dashboards are actionable, (4) Observability Platform — centralized dashboard management with self-service for teams, (5) Training — ensure all operators know how to navigate the dashboard hierarchy."

## Follow-Up Questions

1. "How do you handle dashboards for a system with 100+ microservices?"
2. "What's the difference between Grafana and other visualization tools (Kibana, Datadog)?"
3. "How do you implement dashboard as code with Terraform and Grafana?"
4. "How do you design dashboards for mobile viewing during on-call?"
5. "How do you handle dashboard performance when querying large datasets?"
