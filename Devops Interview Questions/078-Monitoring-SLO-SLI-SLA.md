# 78. Setting Effective SLOs, SLIs, and SLAs

## Scenario

Management wants to define reliability targets for a critical banking application. The application handles 10,000 transactions per minute, processes $50M in daily volume, and must be available 24/7. Currently, there are no formal reliability targets — the team reacts to incidents without clear success criteria. The CTO wants to implement SLOs, SLIs, and SLAs. The application has 15 microservices, runs on Kubernetes in AWS, and uses PostgreSQL databases with cross-region replication. You need to define SLIs, set SLOs, and negotiate SLAs.

## Interviewer Question

"How do you define SLIs, SLOs, and SLAs for a critical banking application that must be available 24/7 with 99.99% uptime?"

## What I Should Think About

- SLI (Service Level Indicator): What you measure
- SLO (Service Level Objective): What you target
- SLA (Service Level Agreement): What you contractually guarantee
- Error budget: 100% - SLO = allowed downtime
- Different SLIs for different aspects (availability, latency, correctness)
- Multi-service vs. system-level SLOs
- Banking compliance requirements (PCI-DSS, SOC2)
- Cost of higher reliability (diminishing returns)

## Ideal Answer

**1. Define SLIs (What to Measure)**
- **Availability SLI**: Percentage of successful requests (HTTP 2xx/3xx)
- **Latency SLI**: Percentage of requests faster than threshold (e.g., 95% < 500ms)
- **Correctness SLI**: Percentage of transactions processed correctly
- **Freshness SLI**: How recent is the data (for read-heavy services)

**2. Set SLOs (What to Target)**
- **Availability SLO**: 99.99% (52 minutes downtime/year)
- **Latency SLO**: 95% of requests < 500ms, 99% < 2s
- **Correctness SLO**: 99.999% of transactions processed correctly
- **Error Budget**: 0.01% = 52 minutes/year of allowed downtime

**3. Negotiate SLAs (What to Contract)**
- SLA should be less stringent than SLO (e.g., 99.95%)
- Include penalties for SLA breaches
- Define measurement methodology
- Exclusions: planned maintenance, force majeure

**4. Implement Monitoring**
- Real-time SLO dashboards
- Error budget burn rate alerts
- Monthly SLO reports for management

## Architecture

```
┌─────────────────────────────────────────────────┐
│         SLO/SLI/SLA FRAMEWORK                    │
│                                                   │
│  ┌──────────────────────────────────────────┐   │
│  │              SLIs (Measures)              │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ │   │
│  │  │Avail.    │ │Latency   │ │Correctness│ │   │
│  │  │99.99%    │ │p95<500ms │ │99.999%   │ │   │
│  │  └────┬─────┘ └────┬─────┘ └────┬─────┘ │   │
│  └───────┼─────────────┼────────────┼───────┘   │
│          ▼             ▼            ▼             │
│  ┌──────────────────────────────────────────┐   │
│  │              SLOs (Targets)               │   │
│  │  ┌──────────────────────────────────┐    │   │
│  │  │ Error Budget: 0.01% = 52 min/yr  │    │   │
│  │  │ Burn Rate Alerts: 14.4x (1hr)    │    │   │
│  │  │                    3x (6hr)      │    │   │
│  │  └──────────────────────────────────┘    │   │
│  └─────────────────────┬───────────────────┘   │
│                        ▼                         │
│  ┌──────────────────────────────────────────┐   │
│  │              SLAs (Contracts)             │   │
│  │  ┌──────────────────────────────────┐    │   │
│  │  │ Availability: 99.95% (SLA)       │    │   │
│  │  │ Penalty: Service credits          │    │   │
│  │  │ Measurement: Monthly              │    │   │
│  │  └──────────────────────────────────┘    │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

## Investigation

1. **Measure current availability:**
   ```promql
   # Current availability over last 30 days
   (
     sum(rate(http_requests_total{status!~"5.."}[30d]))
     /
     sum(rate(http_requests_total[30d]))
   ) * 100
   
   # Availability by service
   (
     sum(rate(http_requests_total{status!~"5.."}[30d])) by (service)
     /
     sum(rate(http_requests_total[30d])) by (service)
   ) * 100
   ```

2. **Measure current latency:**
   ```promql
   # p95 latency over last 30 days
   histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[30d])) by (le))
   
   # p99 latency over last 30 days
   histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[30d])) by (le))
   ```

3. **Calculate error budget:**
   ```promql
   # Error budget remaining (30-day window)
   (
     1 - (
       sum(rate(http_requests_total{status=~"5.."}[30d]))
       /
       sum(rate(http_requests_total[30d]))
     )
   ) / (1 - 0.9999) * 100
   
   # Error budget burn rate (1-hour window)
   (
     sum(rate(http_requests_total{status=~"5.."}[1h]))
     /
     sum(rate(http_requests_total[1h]))
   ) / (1 - 0.9999)
   ```

4. **Check compliance requirements:**
   ```bash
   # PCI-DSS availability requirements
   # PCI-DSS requires 99.9% for cardholder data environments
   # SOC2 requires documented SLAs and monitoring
   
   # Check current SLA against requirements
   echo "Current SLA: 99.95%"
   echo "PCI-DSS requirement: 99.9%"
   echo "SLA meets PCI-DSS: $(echo "99.95 >= 99.9" | bc -l)"
   ```

## Commands

```bash
# 1. Create SLO dashboard in Grafana
# Use grafana-slo-datasource plugin
cat > slo-dashboard.json << 'EOF'
{
  "panels": [
    {
      "title": "Availability SLO",
      "type": "gauge",
      "targets": [{
        "expr": "sum(rate(http_requests_total{status!~\"5..\"}[30d])) / sum(rate(http_requests_total[30d])) * 100",
        "legendFormat": "Availability"
      }],
      "fieldConfig": {
        "defaults": {
          "thresholds": {
            "steps": [
              {"value": 0, "color": "red"},
              {"value": 99.95, "color": "yellow"},
              {"value": 99.99, "color": "green"}
            ]
          }
        }
      }
    }
  ]
}
EOF

# 2. Create error budget alert
cat > slo-alerts.yml << 'EOF'
groups:
  - name: slo-burn-rate
    rules:
      # Fast burn: 14.4x burn rate for 1 hour = 1% budget in 1 hour
      - alert: ErrorBudgetBurnFast
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[1h])) by (service)
            /
            sum(rate(http_requests_total[1h])) by (service)
          ) / (1 - 0.9999) > 14.4
        for: 2m
        labels:
          severity: p1
        annotations:
          summary: "Fast error budget burn for {{ $labels.service }}"
      
      # Slow burn: 3x burn rate for 6 hours = 1% budget in 6 hours
      - alert: ErrorBudgetBurnSlow
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[6h])) by (service)
            /
            sum(rate(http_requests_total[6h])) by (service)
          ) / (1 - 0.9999) > 3
        for: 15m
        labels:
          severity: p2
        annotations:
          summary: "Slow error budget burn for {{ $labels.service }}"
EOF

# 3. Generate monthly SLO report
cat > slo-report.sh << 'EOF'
#!/bin/bash
SERVICE=$1
MONTH=$(date -d "last month" +%Y-%m)

AVAILABILITY=$(curl -s "http://prometheus:9090/api/v1/query" \
  --data-urlencode "query=sum(rate(http_requests_total{service=\"$SERVICE\",status!~\"5..\"}[$MONTH])) / sum(rate(http_requests_total{service=\"$SERVICE\"}[$MONTH])) * 100" \
  | jq -r '.data.result[0].value[1]')

echo "SLO Report for $SERVICE - $MONTH"
echo "Availability: $AVAILABILITY%"
echo "Target: 99.99%"
echo "Error Budget Used: $(echo "scale=2; (100 - $AVAILABILITY) / 0.01" | bc)%"
EOF
```

## Root Cause

| Root Cause | Elimination |
|---|---|
| No defined reliability targets | Implement SLO framework |
| No error budget tracking | Real-time error budget dashboards |
| SLAs not aligned with business needs | Negotiate SLAs with stakeholders |
| No alerting on SLO breaches | Burn rate alerts |
| No measurement methodology | Standardize SLI calculation |

## Immediate Mitigation

1. **Define SLIs** based on current monitoring data
2. **Set initial SLOs** based on historical performance
3. **Create SLO dashboard** in Grafana
4. **Configure burn rate alerts** for SLO breaches

## Permanent Fix

1. Implement SLO framework with error budgets
2. Monthly SLO reports for management
3. Burn rate alerts for proactive detection
4. SLA negotiation with clear measurement methodology
5. SLO review and adjustment process (quarterly)

## Monitoring

- **Real-time SLO dashboards** — availability, latency, correctness
- **Error budget burn rate** — fast (14.4x) and slow (3x) burn alerts
- **Monthly SLO reports** — trend analysis and forecasting
- **SLA compliance tracking** — ensure contractual obligations met

## Security

- SLO data may be sensitive (business-critical metrics)
- Restrict SLO dashboard access to authorized personnel
- SLA contracts should be confidential
- SLO data should be tamper-proof (immutable storage)

## Production Considerations

- **99.99% uptime** = 52 minutes/year downtime — very strict
- Cost of higher reliability increases exponentially
- Consider multi-region deployment for high availability
- Banking compliance requires documented SLAs and monitoring
- Error budgets should drive deployment velocity (burn budget = ship faster)

## Senior-Level Answer

"I'd implement a 3-layer SLO framework: (1) SLIs — measure what matters (availability, latency, correctness), (2) SLOs — set targets based on business requirements (99.99% for banking), (3) SLAs — negotiate contracts with penalties. The key insight is that SLOs should drive engineering decisions — if error budget is healthy, ship faster; if depleted, focus on reliability."

## Architect-Level Answer

"At the organizational level, I'd establish: (1) SLO Governance — quarterly review with business stakeholders, (2) Error Budget Policy — define what happens when budget is depleted (feature freeze, reliability sprint), (3) SLA Framework — standardize SLA structure across services, (4) Compliance Integration — map SLOs to PCI-DSS/SOC2 requirements, (5) Cost-Reliability Tradeoff — model cost of higher reliability for business decisions."

## Follow-Up Questions

1. "How do you handle SLOs for a system with no historical data?"
2. "What's the difference between composite SLOs and per-service SLOs?"
3. "How do you implement SLOs for async processing (Kafka, SQS)?"
4. "How do you handle SLOs across multiple cloud providers?"
5. "How do you communicate SLO status to non-technical stakeholders?"
