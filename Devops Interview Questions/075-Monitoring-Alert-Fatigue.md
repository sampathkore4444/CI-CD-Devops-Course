# 75. Alert Fatigue - Too Many False Positives

## Scenario

The operations team receives 200+ alerts per day across PagerDuty and Slack. Of these, approximately 60% are false positives or low-severity alerts that don't require action. Critical alerts — like database connection pool exhaustion — get buried in the noise. Two weeks ago, a P1 database outage was missed because the critical alert appeared between 50 low-severity alerts. The team has started ignoring alerts entirely, manually checking dashboards instead. You need to redesign the alerting strategy from scratch.

## Interviewer Question

"Our team receives 200+ alerts per day with 60% false positives. Critical alerts are being missed. How do you redesign our alerting strategy to reduce noise while ensuring critical issues are caught?"

## What I Should Think About

- Alert fatigue is an organizational risk, not just a technical problem
- Need to categorize alerts by severity (P1-P4)
- Need to differentiate between symptoms and causes
- Multi-window, multi-burn-rate alerting (Google SRE approach)
- Alert deduplication and grouping
- Escalation policies and on-call rotations
- Alert thresholds based on SLOs, not arbitrary numbers
- Runbook automation for common alerts
- Regular alert review and cleanup

## Ideal Answer

**1. Audit Current Alerts**
- Catalog all 200+ alerts with severity, frequency, and actionability
- Identify alerts that have never fired or always resolve automatically
- Identify alerts that fire during normal operations (false positives)

**2. Implement Alert Tiers**
- **P1 (Critical)**: Page immediately, requires human action within 5 minutes
- **P2 (High)**: Page during business hours, action within 30 minutes
- **P3 (Medium)**: Slack notification, action within 4 hours
- **P4 (Low)**: Dashboard only, review during business hours

**3. Multi-Window Burn Rate Alerting**
- Instead of "CPU > 90% for 5 minutes", use SLO-based alerting
- Calculate burn rate: how fast are we consuming error budget?
- Use multiple time windows (short: 5m, long: 1h) to reduce false positives

**4. Alert Suppression and Grouping**
- Suppress low-severity alerts during maintenance windows
- Group related alerts (e.g., all pod restart alerts for same service)
- Deduplicate alerts within a time window

**5. Runbook Automation**
- Attach runbooks to every alert
- Auto-resolve alerts that self-correct within threshold
- Auto-remediate common issues (pod restarts, disk cleanup)

**6. Regular Review**
- Monthly alert review meeting
- Track alert-to-incident ratio
- Remove or modify alerts that don't lead to action

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│              REDESIGNED ALERTING STRATEGY                 │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │                ALERT SOURCES                      │   │
│  │  Prometheus │ CloudWatch │ Custom Scripts │ Logs  │   │
│  └─────────────────────┬────────────────────────────┘   │
│                        ▼                                 │
│  ┌──────────────────────────────────────────────────┐   │
│  │              ALERT PROCESSING                     │   │
│  │  ┌────────────┐ ┌──────────┐ ┌────────────────┐  │   │
│  │  │ Dedup      │ │ Group    │ │ Severity       │  │   │
│  │  │ (5m window)│ │ (by svc) │ │ Classification │  │   │
│  │  └────────────┘ └──────────┘ └────────────────┘  │   │
│  └─────────────────────┬────────────────────────────┘   │
│                        ▼                                 │
│  ┌──────────────────────────────────────────────────┐   │
│  │              ROUTING BY SEVERITY                   │   │
│  │                                                   │   │
│  │  P1 ──► PagerDuty (Immediate Page)                │   │
│  │  P2 ──► PagerDuty (Business Hours)                │   │
│  │  P3 ──► Slack #incidents (Notification)           │   │
│  │  P4 ──► Slack #monitoring (Dashboard Only)        │   │
│  └─────────────────────┬────────────────────────────┘   │
│                        ▼                                 │
│  ┌──────────────────────────────────────────────────┐   │
│  │              ESCALATION POLICY                    │   │
│  │  L1: On-call Engineer (5 min)                     │   │
│  │  L2: Senior Engineer (15 min)                     │   │
│  │  L3: Engineering Manager (30 min)                 │   │
│  │  L4: VP Engineering (1 hour)                      │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

## Investigation

1. **Audit existing alerts:**
   ```bash
   # Export all Prometheus alerting rules
   curl -s http://prometheus:9090/api/v1/rules | jq '.data.groups[].rules[] | select(.type == "alerting") | {name: .name, severity: .labels.severity, query: .query}'
   
   # Count alerts by severity
   curl -s http://prometheus:9090/api/v1/alerts | jq '.data.alerts[] | .labels.severity' | sort | uniq -c | sort -rn
   ```

2. **Check alert history for false positives:**
   ```promql
   # Alerts that resolved automatically within 5 minutes
   count(
     changes(alertmanager_alerts_resolved_total[1h]) > 5
   ) by (alertname)
   
   # Alerts that fire most frequently
   topk(10, sum(increase(prometheus_rule_evaluation_duration_seconds_total[24h])) by (alertname))
   ```

3. **Measure alert fatigue metrics:**
   ```promql
   # Average alerts per day
   avg_over_time(sum(increase(prometheus_notifications_total[24h]))[7d:24h])
   
   # False positive rate (alerts that resolve within 5 min)
   count(ALERTS{alertstate="firing"} unless ALERTS{alertstate="firing"} offset 5m) / count(ALERTS{alertstate="firing"})
   ```

## Commands

```bash
# 1. Create burn rate alerting rule
cat > burn-rate-alert.yml << 'EOF'
groups:
  - name: slo-burn-rate
    rules:
      # 14.4x burn rate = 1% error budget consumed in 1 hour
      - alert: HighErrorBurnRate
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[1h])) by (service)
            /
            sum(rate(http_requests_total[1h])) by (service)
          ) > 0.144
        for: 2m
        labels:
          severity: p1
        annotations:
          summary: "High error burn rate for {{ $labels.service }}"
          runbook: "https://wiki/runbooks/high-error-burn-rate"
      
      # 3x burn rate = slow burn over 6 hours
      - alert: SlowErrorBurnRate
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[6h])) by (service)
            /
            sum(rate(http_requests_total[6h])) by (service)
          ) > 0.03
        for: 15m
        labels:
          severity: p2
        annotations:
          summary: "Slow error burn rate for {{ $labels.service }}"
          runbook: "https://wiki/runbooks/slow-error-burn"
EOF

# 2. Implement alert grouping in Alertmanager
cat > alertmanager.yml << 'EOF'
global:
  resolve_timeout: 5m

route:
  receiver: default
  group_by: ['alertname', 'service', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - match:
        severity: p1
      receiver: pagerduty-critical
      group_wait: 10s
      repeat_interval: 1h
    - match:
        severity: p2
      receiver: pagerduty-high
      repeat_interval: 4h
    - match:
        severity: p3
      receiver: slack-medium
      repeat_interval: 12h
    - match:
        severity: p4
      receiver: slack-low
      repeat_interval: 24h

receivers:
  - name: pagerduty-critical
    pagerduty_configs:
      - service_key: <key>
        severity: critical
  - name: slack-medium
    slack_configs:
      - channel: '#monitoring'
        send_resolved: true
EOF

# 3. Implement alert suppression during maintenance
cat > suppression-rules.yml << 'EOF'
# Suppress non-P1 alerts during maintenance window
- alert: MaintenanceWindow
  expr: time() > 1705276800 and time() < 1705280400  # 2 hour window
  labels:
    severity: p1
  annotations:
    summary: "Maintenance window active"
EOF

# 4. Track alert metrics
# Add to Prometheus
prometheus_alerts_total{alertname, severity, resolved}
prometheus_alert_duration_seconds{alertname}
prometheus_false_positives_total{alertname}
```

## Root Cause

| Root Cause | Elimination |
|---|---|
| Alerts based on symptoms, not causes | Use SLO-based alerting (burn rates) |
| Arbitrary thresholds (CPU > 90%) | Use error budget-based thresholds |
| No severity classification | Implement P1-P4 severity tiers |
| No alert grouping/deduplication | Configure Alertmanager grouping |
| Alerts without runbooks | Require runbooks for all P1/P2 alerts |
| No regular alert review | Monthly alert review meetings |

## Immediate Mitigation

1. **Disable all P4 alerts immediately** — they provide no value
2. **Group related alerts** in Alertmanager to reduce notification volume
3. **Implement alert suppression** for known maintenance windows
4. **Create runbooks** for top 10 most frequent alerts

## Permanent Fix

1. Implement SLO-based burn rate alerting for all critical services
2. Establish P1-P4 severity classification with clear escalation paths
3. Implement alert deduplication and grouping
4. Create a monthly alert review process
5. Track alert-to-incident ratio as a KPI
6. Implement auto-remediation for common alerts (pod restarts, disk cleanup)

## Monitoring

- **Alert volume trend** — should decrease over time
- **Mean Time to Acknowledge (MTTA)** — should decrease as noise decreases
- **False positive rate** — target < 10%
- **Alert-to-incident ratio** — target > 50% (more alerts lead to incidents)
- **Mean Time to Detect (MTTD)** — should decrease with better alerts

## Security

- Alert notifications should not contain sensitive data
- PagerDuty/Slack access should be restricted to authorized personnel
- Alert rules should be version controlled and reviewed
- Sensitive alerts (security incidents) should have separate escalation paths

## Production Considerations

- **200 alerts/day** → target 20-30 actionable alerts/day
- Alert fatigue is a safety risk — treat it as a production incident
- Consider implementing AIOps for anomaly detection (reduces rules-based alerts)
- Cost: PagerDuty $21/user/month, reducing noise saves on-call fatigue
- Implement alert review as part of sprint retrospective

## Senior-Level Answer

"I'd implement a 4-phase approach: (1) Audit all 200+ alerts and categorize by severity and actionability, (2) Implement SLO-based burn rate alerting to replace arbitrary thresholds, (3) Configure Alertmanager for grouping, deduplication, and routing by severity, (4) Establish a monthly alert review process with clear ownership. The key insight is that alerting should be driven by SLOs — if an alert doesn't represent a risk to user experience or SLO compliance, it shouldn't be an alert."

## Architect-Level Answer

"At an organizational level, I'd establish Observability as a Practice with: (1) Alerting Standards — every team must follow the same severity classification and escalation policy, (2) SLO Framework — define SLOs for all critical services with error budgets, (3) Alert Governance — monthly review with clear ownership and escalation, (4) Tooling Standardization — single alerting platform (Alertmanager + PagerDuty) with consistent configuration, (5) Cultural Shift — from 'alert on everything' to 'alert on what matters' — measured by MTTA and alert-to-incident ratio."

## Follow-Up Questions

1. "How do you implement multi-window burn rate alerting in Prometheus?"
2. "What's the difference between alerting on symptoms vs. causes?"
3. "How do you handle alert routing for a global team across multiple time zones?"
4. "How would you implement auto-remediation for common alerts?"
5. "What metrics do you use to measure the effectiveness of your alerting strategy?"
