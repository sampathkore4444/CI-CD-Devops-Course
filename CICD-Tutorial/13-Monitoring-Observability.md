# 13 — Monitoring & Observability: Prometheus, Grafana, ELK

> **Goal:** Understand how to monitor CI/CD pipelines and production systems in banking.

---

## 🔍 What is Monitoring & Observability?

**Monitoring** tells you **what** is happening (errors, latency, CPU usage). **Observability** tells you **why** it's happening (distributed tracing, log correlation, root cause analysis).

```
Monitoring:  "The payment service has 5% error rate"
Observability: "The payment service has 5% error rate because the 
                database connection pool is exhausted after a 
                spike in concurrent transfers at 2:00 PM"
```

---

## 🏗️ The Three Pillars of Observability

```
┌─────────────────────────────────────────────────────────────┐
│                   OBSERVABILITY                              │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │   METRICS   │  │    LOGS     │  │   TRACES    │        │
│  │             │  │             │  │             │        │
│  │ Numerical   │  │ Textual     │  │ Request     │        │
│  │ data over   │  │ events from │  │ flow across │        │
│  │ time        │  │ components  │  │ services    │        │
│  │             │  │             │  │             │        │
│  │ "What?"     │  │ "Details?"  │  │ "Where?"    │        │
│  │             │  │             │  │             │        │
│  │ Prometheus  │  │ ELK Stack   │  │ Jaeger      │        │
│  │ Grafana     │  │ Fluentd     │  │ Zipkin      │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

---

## 📊 Prometheus + Grafana: Metrics & Dashboards

### What is Prometheus?
Prometheus **scrapes** (collects) metrics from applications and infrastructure at regular intervals and stores them in a time-series database.

### What is Grafana?
Grafana **visualizes** Prometheus metrics in beautiful dashboards.

### Key Metrics (The Four Golden Signals)
```
1. LATENCY     → How long requests take
2. TRAFFIC     → How many requests per second
3. ERRORS      → Rate of failed requests
4. SATURATION  → How "full" the service is (CPU, memory)
```

### Prometheus Configuration
```yaml
# prometheus.yml
global:
  scrape_interval: 15s

rule_files:
  - "alert_rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

scrape_configs:
  - job_name: 'payment-service'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
  
  - job_name: 'kubernetes-nodes'
    static_configs:
      - targets: ['node1:9100', 'node2:9100', 'node3:9100']
```

### Alert Rules
```yaml
# alert_rules.yml
groups:
  - name: banking-alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5..",service="payment"}[5m]))
          / sum(rate(http_requests_total{service="payment"}[5m])) > 0.01
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Payment service error rate > 1%"
          description: "Error rate is {{ $value | humanizePercentage }}"
      
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99, 
            sum(rate(http_request_duration_seconds_bucket{service="payment"}[5m])) by (le)
          ) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Payment P99 latency > 2 seconds"
      
      - alert: PodCrashLooping
        expr: rate(kube_pod_container_status_restarts_total{namespace="production"}[15m]) > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Pod {{ $labels.pod }} is crash looping"
```

---

## 📝 ELK Stack: Centralized Logging

### What is ELK?
- **Elasticsearch** — search and store logs
- **Logstash** — process and transform logs
- **Kibana** — visualize and search logs

### Log Collection Architecture
```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Payment Pod  │  │ Account Pod  │  │ Notify Pod   │
│ → stdout     │  │ → stdout     │  │ → stdout     │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       ▼                 ▼                 ▼
┌─────────────────────────────────────────────────────┐
│              Fluentd (Log Collector)                  │
│  - Collects logs from all pods                       │
│  - Adds metadata (pod name, namespace, timestamp)    │
│  - Forwards to Elasticsearch                         │
└─────────────────────────┬───────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────┐
│              Elasticsearch                           │
│  - Indexes logs                                      │
│  - Full-text search                                  │
│  - Retention policies (90 days for banking)          │
└─────────────────────────┬───────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────┐
│              Kibana                                  │
│  - Search: "Show me all errors in payment service"   │
│  - Dashboard: Real-time error rate graph             │
│  - Alerts: Notify on error spike                     │
└─────────────────────────────────────────────────────┘
```

---

## 🏦 Real-World Banking Scenarios

### Scenario 1: Real-Time Payment Monitoring Dashboard
**Context:** Operations team needs a live dashboard showing all payment service metrics.

**Grafana Dashboard:**
```
┌─────────────────────────────────────────────────────────┐
│              PAYMENT SERVICE DASHBOARD                    │
│                                                         │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐       │
│  │ Requests/s │  │ Error Rate │  │ P99 Latency│       │
│  │   1,247    │  │   0.002%   │  │   340ms    │       │
│  │   📈 +5%   │  │   ✅ OK    │  │   ✅ OK    │       │
│  └────────────┘  └────────────┘  └────────────┘       │
│                                                         │
│  ┌──────────────────────────────────────────────┐      │
│  │          Request Rate (last 24 hours)         │      │
│  │  2000├─────────────────────────────────────  │      │
│  │  1500├──────────────────────────╱╲──────╱╲   │      │
│  │  1000├────────────────────╱╲───╱──╲───╱──╲  │      │
│  │   500├──────────╱╲───────╱──╲─╱────╲─╱────╲ │      │
│  │      ├────╱╲───╱──╲─────╱────╲──────╲──────│      │
│  │      └───────────────────────────────────────│      │
│  │       00:00  06:00  12:00  18:00  24:00     │      │
│  └──────────────────────────────────────────────┘      │
│                                                         │
│  ┌─────────────────────┐  ┌─────────────────────┐     │
│  │  Error Rate Trend   │  │  Latency Distribution│     │
│  │  (last 6 hours)     │  │                     │     │
│  │  0.1%├──────         │  │  <100ms  ████████  │     │
│  │  0.0%├──────         │  │  100-500ms████     │     │
│  │      └────────────── │  │  >500ms  █         │     │
│  └─────────────────────┘  └─────────────────────┘     │
└─────────────────────────────────────────────────────────┘
```

### Scenario 2: CI/CD Pipeline Monitoring
**Context:** Track pipeline health, build times, and failure rates.

**Pipeline Metrics:**
```yaml
# Custom metrics exposed by Jenkins/GitLab
pipeline_duration_seconds{stage="build",status="success"} 45
pipeline_duration_seconds{stage="test",status="success"} 180
pipeline_duration_seconds{stage="deploy",status="success"} 60
pipeline_total{status="success"} 150
pipeline_total{status="failed"} 5
pipeline_failure_rate 0.033  # 3.3% failure rate
```

**Pipeline Dashboard:**
```
┌─────────────────────────────────────────────────────────┐
│              CI/CD PIPELINE DASHBOARD                     │
│                                                         │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐       │
│  │ Builds/Day │  │ Success %  │  │ Avg Build  │       │
│  │    47      │  │   96.8%    │  │   8m 32s   │       │
│  │   📈 +12%  │  │   ✅ OK    │  │   📈 -15%  │       │
│  └────────────┘  └────────────┘  └────────────┘       │
│                                                         │
│  ┌──────────────────────────────────────────────┐      │
│  │     Build Duration by Service (last 7 days)   │      │
│  │                                              │      │
│  │  payment-service   ████████ 8m               │      │
│  │  account-service   ██████ 6m                 │      │
│  │  notification-svc  ████ 4m                   │      │
│  │  gateway-service   ████████████ 12m          │      │
│  │  reporting-service ██████ 6m                 │      │
│  └──────────────────────────────────────────────┘      │
│                                                         │
│  Recent Failures:                                       │
│  ❌ payment-service #1847 - Unit test failure (2h ago)  │
│  ❌ gateway-service #923 - Security scan failed (5h ago)│
│  ✅ All other builds successful                         │
└─────────────────────────────────────────────────────────┘
```

### Scenario 3: Distributed Tracing for Payment Flows
**Context:** A UPI payment takes 8 seconds. Need to find which service is slow.

**Trace Waterfall:**
```
Trace ID: tx-abc123def456
Duration: 8,234ms

┌─────────────────────────────────────────────────────────┐
│ API Gateway (50ms)                                      │
│ └─ Payment Service (7,500ms) ⚠️ SLOW                    │
│    ├─ Validate Account (100ms) ✅                        │
│    ├─ Check Balance (120ms) ✅                           │
│    ├─ Fraud Detection (5,000ms) ⚠️ SLOW                 │
│    │  └─ ML Model Inference (4,900ms) ⚠️ SLOW           │
│    ├─ Debit Account (200ms) ✅                           │
│    ├─ Credit Account (180ms) ✅                          │
│    └─ Send Notification (500ms) ✅                       │
│                                                         │
│ ROOT CAUSE: ML model inference taking 4.9 seconds       │
│ RECOMMENDATION: Optimize model or add caching           │
└─────────────────────────────────────────────────────────┘
```

---

## 🏦 Banking End-to-End Examples

### E2E Example 1: Complete Payment Service Monitoring Stack

**Context:** Set up end-to-end monitoring for payment service handling 10 million transactions/day.

```yaml
# Prometheus configuration
# File: prometheus/prometheus.yml

scrape_configs:
  - job_name: 'payment-service'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
    scrape_interval: 15s

# Grafana dashboard
# File: grafana/dashboards/payment-service.json
# Panels:
# 1. Request Rate (req/s)
# 2. Error Rate (%)
# 3. P50/P95/P99 Latency
# 4. Transaction Volume
# 5. Success Rate
# 6. Active Connections
```

```bash
# Deploy monitoring stack
$ kubectl apply -f monitoring/namespace.yaml
$ kubectl apply -f monitoring/prometheus.yaml -n monitoring
$ kubectl apply -f monitoring/grafana.yaml -n monitoring
$ kubectl apply -f monitoring/alertmanager.yaml -n monitoring

# Verify monitoring is running
$ kubectl get pods -n monitoring
# NAME                           READY   STATUS    AGE
# prometheus-abc123              1/1     Running   5m
# grafana-def456                 1/1     Running   5m
# alertmanager-ghi789            1/1     Running   5m
# node-exporter-jkl012           1/1     Running   5m

# Query metrics
$ curl -s 'http://prometheus:9090/api/v1/query?query=rate(http_requests_total{service="payment"}[5m])'
# {"status":"success","data":{"result":[["1693819200","1247.5"]]}}
# 1,247 requests per second ✅

# Alert rules active
$ curl -s 'http://alertmanager:9093/api/v2/alerts'
# [] (no active alerts) ✅
```

### E2E Example 2: Distributed Tracing for Transaction Debugging

**Context:** Customer complains: "My transfer took 30 seconds!" Find the root cause.

```bash
# Customer provides transaction ID: TXN-2026-09-04-abc123

# Step 1: Search in Jaeger
$ curl -s 'http://jaeger:16686/api/traces/TXN-2026-09-04-abc123'

# Trace waterfall:
# Trace ID: TXN-2026-09-04-abc123
# Duration: 32,450ms (32 seconds!)
#
# ┌─────────────────────────────────────────────────────────────────┐
# │ API Gateway (120ms)                                            │
# │ └─ Payment Service (31,800ms) ⚠️ SLOW                          │
# │    ├─ Validate Account (85ms) ✅                                │
# │    ├─ Check Balance (95ms) ✅                                   │
# │    ├─ Fraud Detection (28,000ms) ⚠️ VERY SLOW                  │
# │    │  └─ ML Model Inference (27,500ms) ⚠️ SLOW                 │
# │    │     └─ GPU Queue Wait (25,000ms) ⚠️ BOTTLENECK             │
# │    ├─ Debit Account (150ms) ✅                                  │
# │    ├─ Credit Account (130ms) ✅                                 │
# │    └─ Send Notification (50ms) ✅                               │
# └─────────────────────────────────────────────────────────────────┘
#
# ROOT CAUSE: GPU queue wait (25 seconds)
# ML model inference was fast (2.5 seconds)
# But 50+ transactions were queued waiting for GPU
#
# SOLUTION: Increase GPU instances from 2 to 4
# OR: Implement model caching for frequent patterns

# Step 2: Check logs for correlation
$ kubectl logs -l app=payment -n production --tail=1000 | grep TXN-2026-09-04-abc123
# 2026-09-04 14:30:00 INFO  PaymentService - TXN-2026-09-04-abc123 Starting
# 2026-09-04 14:30:00 INFO  FraudDetection - TXN-2026-09-04-abc123 Queued for ML inference
# 2026-09-04 14:30:25 INFO  FraudDetection - TXN-2026-09-04-abc123 GPU available
# 2026-09-04 14:30:28 INFO  FraudDetection - TXN-2026-09-04-abc123 ML inference complete
# 2026-09-04 14:30:28 INFO  PaymentService - TXN-2026-09-04-abc123 Completed

# Step 3: Fix
$ kubectl scale deployment fraud-detection-gpu --replicas=4 -n production
# GPU instances: 2 → 4
# Queue wait time: 25s → 0.5s ✅
```

### E2E Example 3: SLA Monitoring & Reporting

**Context:** Bank must report SLA compliance to regulators quarterly.

```yaml
# SLA monitoring configuration
# File: monitoring/sla-monitoring.yaml

sla_targets:
  payment-service:
    availability: 99.99%
    latency_p99: 1000ms
    error_rate: 0.01%
    
  account-service:
    availability: 99.99%
    latency_p99: 500ms
    error_rate: 0.01%
    
  loan-service:
    availability: 99.95%
    latency_p99: 2000ms
    error_rate: 0.05%
```

```bash
# Query SLA metrics for last quarter
$ curl -s 'http://prometheus:9090/api/v1/query?query=1 - avg_over_time(http_requests_total{service="payment",status!~"5.."}[90d]) / avg_over_time(http_requests_total{service="payment"}[90d])'
# Result: 0.00012 (99.988% availability)
# SLA Target: 99.99%
# SLA MET: ✅ (0.002% buffer)

# Generate compliance report
$ python scripts/generate-sla-report.py --period q1-2026

# ╔═══════════════════════════════════════════════════════════════════╗
# ║                    Q1 2026 SLA COMPLIANCE REPORT                 ║
# ╠═══════════════════════════════════════════════════════════════════╣
# ║ Service          Availability  Latency P99  Error Rate  Status   ║
# ╠═══════════════════════════════════════════════════════════════════╣
# ║ payment-service  99.988%       340ms        0.002%      ✅ PASS  ║
# ║ account-service  99.995%       180ms        0.001%      ✅ PASS  ║
# ║ loan-service     99.962%       890ms        0.008%      ✅ PASS  ║
# ║ gateway-service  99.999%       45ms         0.0001%     ✅ PASS  ║
# ╠═══════════════════════════════════════════════════════════════════╣
# ║ Overall SLA Compliance: 100% ✅                                   ║
# ║ Total Downtime: 12 minutes 45 seconds                            ║
# ║ Incidents: 2 (both resolved within SLA)                          ║
# ╚═══════════════════════════════════════════════════════════════════╝
```

---

## 📋 Interview Questions

### Q1: What is the difference between monitoring and observability?
**Answer:** Monitoring tells you **what** is broken (error rate is high, latency is slow). Observability tells you **why** it's broken (distributed tracing shows which service is slow, logs show the specific error). Monitoring is a subset of observability. In banking, you need both: monitoring for alerting, observability for debugging.

### Q2: What are the Four Golden Signals and why do they matter?
**Answer:** 

(1) **Latency** — time to serve a request. 

(2) **Traffic** — demand on the system. 

(3) **Errors** — rate of failed requests. 

(4) **Saturation** — how "full" resources are. 

These four signals give a complete health picture. 

For banking: latency affects customer experience, traffic affects capacity planning, errors affect revenue, saturation predicts outages.

### Q3: How do you set up meaningful alerts without alert fatigue?
**Answer:** Alert fatigue happens when too many low-priority alerts desensitize the team. 

Strategies: 

(1) **Symptom-based alerts** — alert on user impact (error rate > 1%), not causes (CPU > 80%). 

(2) **Severity levels** — P1 (page immediately), P2 (Slack notification), P3 (email). 

(3) **Alert on trends** — not instant spikes. 

(4) **Runbooks** — every alert should have a documented response. 

(5) **Regular review** — remove alerts that never fire or never matter.

### Q4: How do you correlate logs, metrics, and traces?
**Answer:** Use a **trace ID** propagated across all three: 

(1) **Metrics** — aggregate by trace ID to see which transactions are slow. 

(2) **Logs** — include trace ID in log lines to search logs for a specific request. 

(3) **Traces** — follow the request across services. 

Tools like Grafana Tempo + Loki + Prometheus provide unified observability. 

In banking, this is essential for debugging payment failures.

### Q5: What SLIs, SLOs, and SLAs should a banking payment service have?
**Answer:** 

**SLIs** (Service Level Indicators): error rate, latency, throughput. 

**SLOs** (Service Level Objectives): error rate < 0.1%, P99 latency < 1s, availability > 99.99%. 

**SLAs** (Service Level Agreements): contractual guarantees (99.99% uptime = 52 minutes downtime/year). 

Banking typically requires 99.999% availability for core payment services (5.26 minutes downtime/year).

---

## 📚 Summary

| Concept | Key Takeaway |
|---------|-------------|
| Metrics | Numerical data (Prometheus) |
| Logs | Textual events (ELK/Fluentd) |
| Traces | Request flow across services (Jaeger) |
| Grafana | Visualize metrics in dashboards |
| Alerts | Notify on issues before customers notice |
| Banking Relevance | Real-time monitoring, compliance, debugging |

**Next:** [14-Security-in-CICD.md](./14-Security-in-CICD.md) — Learn DevSecOps and security in CI/CD.
