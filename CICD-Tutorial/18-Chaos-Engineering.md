# 18 — Chaos Engineering: Testing Resilience

> **Goal:** Understand Chaos Engineering — how to proactively test system resilience by injecting controlled failures.

---

## 📑 Table of Contents

- [🔍 What is Chaos Engineering?](#-what-is-chaos-engineering)
- [🛠️ Chaos Engineering Tools](#️-chaos-engineering-tools)
- [🏗️ Chaos Engineering Architecture](#️-chaos-engineering-architecture)
- [🏦 Banking End-to-End Examples](#-banking-end-to-end-examples)
  - [E2E Example 1: Pod Failure Test](#e2e-example-1-pod-failure-test)
  - [E2E Example 2: Network Partition Test](#e2e-example-2-network-partition-test)
  - [E2E Example 3: Database Failover Test](#e2e-example-3-database-failover-test)
- [📋 Interview Questions](#-interview-questions)
- [📚 Summary](#-summary)

---

## 🔍 What is Chaos Engineering?

**Chaos Engineering** is the practice of **intentionally injecting failures** into a system to discover weaknesses before they cause real outages.

### The Chaos Engineering Principles

```
1. BUILD A HYPOTHESIS    → "What do we think will happen?"
2. INTRODUCE REAL EVENTS  → Kill pods, inject latency, fill disk
3. OBSERVE THE DAMAGE     → Monitor metrics, logs, traces
4. MINIMIZE BLAST RADIUS  → Start small, contain failures
5. AUTOMATE IN PRODUCTION → Regular game days, not one-time
```

### Why Banks Need Chaos Engineering

```
Without Chaos:
  "Everything works in testing"
  → Production outage on salary day
  → 1 million customers affected
  → ₹50 crore loss
  → Regulatory inquiry

With Chaos:
  "We intentionally broke things in staging"
  → Found database connection pool exhaustion
  → Fixed before production
  → Salary day: 0 issues
  → Customers happy, regulators satisfied
```

---

## 🛠️ Chaos Engineering Tools

| Tool | What It Does | Banking Use Case |
|------|-------------|------------------|
| **Litmus** | Kubernetes-native chaos | Pod kill, network partition |
| **Chaos Monkey** | Random instance termination | Test auto-scaling |
| **Gremlin** | Enterprise chaos platform | Full-scale failure injection |
| **Chaos Toolkit** | Declarative chaos experiments | Automated resilience testing |
| **AWS Fault Injection** | Cloud-native chaos | AWS service failures |

---

## 🏗️ Chaos Engineering Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CHAOS ENGINEERING WORKFLOW                        │
│                                                                     │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐          │
│  │  HYPOTHESIS │────▶│   INJECT    │────▶│  OBSERVE    │          │
│  │             │     │   FAILURE   │     │  IMPACT     │          │
│  │ "If we kill │     │             │     │             │          │
│  │  payment pod│     │ kubectl     │     │ Monitor:    │          │
│  │  K8s will   │     │ delete pod  │     │ - Error rate│          │
│  │  restart it │     │             │     │ - Latency   │          │
│  │  in <30s"   │     │             │     │ - Throughput│          │
│  └─────────────┘     └─────────────┘     └──────┬──────┘          │
│                                                  │                  │
│                                                  ▼                  │
│                                          ┌─────────────┐           │
│                                          │   ANALYZE   │           │
│                                          │             │           │
│                                          │ Hypothesis  │           │
│                                          │ confirmed?  │           │
│                                          │             │           │
│                                          │ YES: System │           │
│                                          │ resilient ✅ │           │
│                                          │             │           │
│                                          │ NO: Fix     │           │
│                                          │ weakness 🔧 │           │
│                                          └─────────────┘           │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🏦 Banking End-to-End Examples

### E2E Example 1: Pod Failure Test

**Context:** Test if payment service survives pod crashes.

```yaml
# Chaos experiment
# File: chaos/pod-kill-experiment.yaml

apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: payment-pod-kill
  namespace: production
spec:
  appinfo:
    appns: production
    applabel: app=payment-service
    appkind: deployment
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: '300'  # 5 minutes
            - name: CHAOS_INTERVAL
              value: '60'   # Kill pod every 60 seconds
            - name: FORCE
              value: 'false'
```

```bash
# Run experiment
$ kubectl apply -f chaos/pod-kill-experiment.yaml
$ kubectl get chaosengine payment-pod-kill -n production
# NAME                  APP-SPEC   STATUS
# payment-pod-kill      Ready      Running

# Monitor during chaos
$ kubectl logs -l app=payment-service -n production --tail=50 | grep -E "ERROR|WARN|killed"
# [Chaos] Pod payment-service-abc123 killed at 10:00:15
# [Recovery] New pod payment-service-xyz789 starting...
# [Recovery] Pod payment-service-xyz789 ready at 10:00:28 (13 seconds)
# [Chaos] Pod payment-service-def456 killed at 10:01:15
# [Recovery] New pod payment-service-uvw901 starting...
# [Recovery] Pod payment-service-uvw901 ready at 10:01:29 (14 seconds)

# Results:
# ✅ Average recovery time: 13.5 seconds (threshold: 30 seconds)
# ✅ Zero failed requests during recovery
# ✅ No cascade failures to other services
# Hypothesis CONFIRMED: System is resilient to pod failures
```

### E2E Example 2: Network Partition Test

**Context:** Test if payment service handles network isolation from database.

```yaml
# Network chaos experiment
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: payment-network-partition
  namespace: production
spec:
  experiments:
    - name: pod-network-loss
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: '120'  # 2 minutes
            - name: NETWORK_INTERFACE
              value: 'eth0'
            - name: NETWORK_PACKET_LOSS_PERCENTAGE
              value: '100'  # Complete network loss
            - name: CONTAINER_RUNTIME
              value: 'containerd'
            - name: TARGET_PODS
              value: 'app=payment-service'
            - name: DESTINATION_IPS
              value: '10.0.1.100'  # Database IP
```

```bash
# Run experiment
$ kubectl apply -f chaos/network-partition-experiment.yaml

# Monitor payment service behavior
$ kubectl logs -l app=payment-service -n production --tail=100
# [10:00:00] Network partition started (100% packet loss to DB)
# [10:00:05] Database connection pool exhausted
# [10:00:06] Circuit breaker OPEN for database connection
# [10:00:06] Serving from Redis cache (stale data, <1s old)
# [10:00:07] New transactions queued for retry
# [10:00:30] Cache hit rate: 85% (degraded but functional)
# [10:01:00] Cache hit rate: 92% (warming up)
# [10:02:00] Network partition ended
# [10:02:01] Database connection restored
# [10:02:02] Processing queued transactions
# [10:02:05] All queued transactions processed
# [10:02:06] System fully recovered ✅

# Results:
# ✅ Service remained available during partition (served from cache)
# ✅ Queued transactions processed after recovery
# ✅ Zero data loss
# ✅ Recovery time: 6 seconds
# Hypothesis CONFIRMED: System handles network partitions gracefully
```

### E2E Example 3: Database Failover Test

**Context:** Test PostgreSQL primary database failover to standby.

```yaml
# Database failover experiment
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: postgres-failover
  namespace: production
spec:
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: '60'
            - name: TARGET_PODS
              value: 'app=postgres,role=primary'
            - name: FORCE
              value: 'true'  # Force kill primary
```

```bash
# Run experiment
$ kubectl apply -f chaos/postgres-failover-experiment.yaml

# Monitor failover
$ kubectl logs -l app=postgres -n production --tail=50
# [10:00:00] Primary database pod killed
# [10:00:01] Replication lag detected
# [10:00:02] Standby promoted to primary
# [10:00:03] New primary accepting connections
# [10:00:04] All replicas re-synced

# Monitor payment service during failover
$ kubectl logs -l app=payment-service -n production --tail=50
# [10:00:00] Database connection lost
# [10:00:01] Retrying connection (attempt 1/3)
# [10:00:02] Retrying connection (attempt 2/3)
# [10:00:03] Connection established to new primary
# [10:00:04] Processing transactions normally

# Results:
# ✅ Failover completed in 3 seconds
# ✅ Payment service reconnected in 3 seconds
# ✅ Zero failed transactions
# ✅ Zero data loss (async replication caught up)
# Hypothesis CONFIRMED: Database failover works correctly
```

---

## 📋 Interview Questions

### Q1: What is the difference between Chaos Engineering and testing?
**Answer:** **Testing** verifies known scenarios against expected outcomes. **Chaos Engineering** explores unknown failure modes by injecting realistic failures. Testing asks: "Does this work?" Chaos asks: "What breaks this?" In banking, testing catches bugs; chaos engineering catches infrastructure weaknesses that tests miss (network partitions, disk failures, cascading failures).

### Q2: How do you minimize risk during chaos experiments?
**Answer:** (1) **Start in staging** — never chaos in production first. (2) **Small blast radius** — kill 1 pod, not all pods. (3) **Time-boxed** — experiments run for 5-10 minutes max. (4) **Automated rollback** — if metrics degrade beyond threshold, stop experiment. (5) **Business hours** — run during low-traffic periods. (6) **Notify teams** — Slack alert before chaos starts. (7) **Have a kill switch** — `kubectl delete chaosengine` stops everything.

### Q3: What is a "Game Day" and how do banks run them?
**Answer:** A Game Day is a **scheduled chaos engineering exercise** where the entire team simulates production failures. Bank Game Day process: (1) **Plan** — define scenarios (pod failure, DB failover, network partition). (2) **Notify** — alert all stakeholders 24 hours before. (3) **Execute** — run experiments during business hours with monitoring. (4) **Observe** — watch dashboards, track recovery times. (5) **Debrief** — document findings, create action items. Frequency: monthly for critical systems, quarterly for all systems.

### Q4: What metrics should you track during chaos experiments?
**Answer:** (1) **Availability** — did the service stay up? (2) **Latency** — did response times degrade? (3) **Error rate** — how many requests failed? (4) **Recovery time** — how long to return to normal? (5) **Data integrity** — any data loss or corruption? (6) **Cascade effects** — did failure spread to other services? (7) **Resource utilization** — CPU/memory spikes during recovery?

### Q5: How do you implement chaos engineering in a CI/CD pipeline?
**Answer:** (1) **Automated resilience tests** — run chaos experiments in staging after deployment. (2) **Chaos gate** — deployment blocked if resilience tests fail. (3) **Scheduled experiments** — weekly chaos against production (low-risk). (4) **Chaos as code** — define experiments in Git, version-controlled. (5) **Metrics-driven** — automatically stop experiment if error rate > threshold.

---

## 📚 Summary

| Concept | Key Takeaway |
|---------|-------------|
| Chaos Engineering | Intentionally inject failures to find weaknesses |
| Hypothesis | "What do we think will happen?" |
| Blast Radius | Start small, contain failures |
| Game Day | Scheduled team chaos exercises |
| Automation | Chaos in CI/CD pipeline |
| Banking Resilience | Proactively test before production failures |

**Next:** [19-Secret-Management.md](./19-Secret-Management.md) — Learn HashiCorp Vault for banking secrets.
