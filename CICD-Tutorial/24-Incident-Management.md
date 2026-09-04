# 24 — Incident Management: When Things Go Wrong

> **Goal:** Master incident response — how banks detect, respond to, and learn from production incidents.

---

## 🔍 What is Incident Management?

**Incident Management** is the process of identifying, triaging, resolving, and learning from production incidents.

### Incident Severity Levels

| Severity | Definition | Response Time | Example |
|----------|-----------|---------------|---------|
| **P1 (Critical)** | Service down, data loss | 15 minutes | Payment system outage |
| **P2 (High)** | Service degraded | 1 hour | Slow transaction processing |
| **P3 (Medium)** | Partial impact | 4 hours | One feature unavailable |
| **P4 (Low)** | Minimal impact | 24 hours | Cosmetic issue |

---

## 🏗️ Incident Response Workflow

```
┌─────────────────────────────────────────────────────────────────────┐
│                    INCIDENT RESPONSE WORKFLOW                        │
│                                                                     │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐          │
│  │  DETECT     │────▶│  TRIAGE     │────▶│  MITIGATE   │          │
│  │             │     │             │     │             │          │
│  │ Monitoring  │     │ Severity    │     │ Rollback    │          │
│  │ alerts      │     │ assignment  │     │ Failover    │          │
│  │ Customer    │     │ War room    │     │ Hotfix      │          │
│  │ reports     │     │ opened      │     │             │          │
│  └─────────────┘     └─────────────┘     └──────┬──────┘          │
│                                                  │                  │
│                                                  ▼                  │
│                                          ┌─────────────┐           │
│                                          │  RESOLVE    │           │
│                                          │             │           │
│                                          │ Fix deployed│           │
│                                          │ Service     │           │
│                                          │ restored    │           │
│                                          └──────┬──────┘           │
│                                                  │                  │
│                                                  ▼                  │
│                                          ┌─────────────┐           │
│                                          │  POST-MORTEM│           │
│                                          │             │           │
│                                          │ Root cause  │           │
│                                          │ analysis    │           │
│                                          │ Action items│           │
│                                          └─────────────┘           │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 📋 Incident Response Runbook

```yaml
# runbooks/payment-outage.yaml

incident:
  name: Payment Service Outage
  severity: P1
  response_time: 15 minutes
  
  detection:
    - alert: "Payment service error rate > 5%"
    - alert: "Payment service latency > 5s"
    - customer_report: "Cannot make payments"
    
  immediate_actions:
    - step: "Check payment service health"
      command: "kubectl get pods -l app=payment -n production"
      expected: "All pods Running"
      
    - step: "Check recent deployments"
      command: "kubectl rollout history deployment/payment -n production"
      expected: "No recent deployments"
      
    - step: "Check database connectivity"
      command: "kubectl exec -it payment-abc123 -- curl -s http://localhost:8080/actuator/health"
      expected: "Status: UP"
      
    - step: "Check Redis cache"
      command: "kubectl exec -it payment-abc123 -- redis-cli -h redis.bank.com ping"
      expected: "PONG"
      
  mitigation:
    - step: "If recent deployment caused issue"
      action: "Rollback to previous version"
      command: "kubectl rollout undo deployment/payment -n production"
      
    - step: "If database is down"
      action: "Failover to standby"
      command: "kubectl patch svc postgres -p '{\"spec\":{\"selector\":{\"role\":\"standby\"}}}'"
      
    - step: "If high traffic causing overload"
      action: "Scale up payment service"
      command: "kubectl scale deployment/payment --replicas=12 -n production"
      
  resolution:
    - step: "Verify service is healthy"
      command: "curl -s https://api.bank.com/api/v1/payments/health"
      expected: "Status: healthy"
      
    - step: "Monitor for 15 minutes"
      command: "watch kubectl logs -l app=payment -n production --tail=10"
      
  post_mortem:
    required: true
    deadline: "72 hours after incident"
    attendees: ["On-call engineer", "Tech lead", "Product owner", "SRE"]
```

---

## 🏦 Banking End-to-End Examples

### E2E Example 1: Payment Service Outage Response

**Context:** Payment service goes down during peak hours (salary day).

```bash
# 10:00 AM - Alert received
[PagerDuty] P1: Payment service error rate 12% (threshold: 1%)
[Slack #incidents] 🔴 Payment service outage detected

# 10:02 AM - Incident commander joins
$ kubectl get pods -l app=payment -n production
# NAME                    READY   STATUS    RESTARTS
# payment-abc123          0/1     CrashLoopBackOff   5
# payment-def456          0/1     CrashLoopBackOff   5
# payment-ghi789          0/1     CrashLoopBackOff   5

# 10:05 AM - Root cause identified
$ kubectl logs payment-abc123 --previous | tail -20
# java.lang.OutOfMemoryError: Java heap space
#   at com.bank.payment.TransferService.processTransfer(TransferService.java:142)
# Memory leak in transfer processing!

# 10:08 AM - Immediate mitigation
# Option 1: Scale up (temporary fix)
$ kubectl scale deployment/payment --replicas=12 -n production

# Option 2: Increase memory limit (better fix)
$ kubectl patch deployment payment -n production -p '{"spec":{"template":{"spec":{"containers":[{"name":"payment","resources":{"limits":{"memory":"2Gi"}}}]}}}}'

# 10:12 AM - Service restored
$ kubectl get pods -l app=payment -n production
# NAME                    READY   STATUS    RESTARTS
# payment-abc123          1/1     Running   0
# payment-def456          1/1     Running   0
# payment-ghi789          1/1     Running   0
# payment-new-1           1/1     Running   0
# payment-new-2           1/1     Running   0

# 10:15 AM - Verify
$ curl -s https://api.bank.com/api/v1/payments/health
# {"status":"healthy","error_rate":"0.002%"}

# 10:30 AM - Monitoring stable
# Error rate: 0.002% (normal)
# Latency: 230ms (normal)
# Transactions: Processing normally
```

### E2E Example 2: Database Failover Incident

**Context:** Primary database fails; standby takes over.

```bash
# 02:15 AM - Database alert
[PagerDuty] P1: PostgreSQL primary database unreachable

# 02:17 AM - DBA joins incident
$ psql -h primary.db.bank.com -U admin -c "SELECT 1;"
# psql: error: connection to server timed out

# 02:18 AM - Verify standby
$ psql -h standby.db.bank.com -U admin -c "SELECT 1;"
#  ?column?
# ----------
#         1
# Standby is healthy ✅

# 02:19 AM - Promote standby to primary
$ psql -h standby.db.bank.com -U admin -c "SELECT pg_promote();"
# pg_promote
# -----------
#  t
# Standby promoted to primary ✅

# 02:20 AM - Update application configuration
$ kubectl set env deployment/payment DB_HOST=standby.db.bank.com -n production
$ kubectl rollout restart deployment/payment -n production

# 02:22 AM - Verify
$ kubectl exec -it payment-abc123 -- curl -s http://localhost:8080/actuator/health
# {"status":"UP","components":{"db":{"status":"UP"}}}

# 02:25 AM - Rebuild primary (using pg_basebackup)
$ pg_basebackup -h standby.db.bank.com -D /var/lib/postgresql/data -U replicator -Fp -Xs -P

# 03:00 AM - Primary rebuilt, replication restored
# Total downtime: 7 minutes
# Zero data loss (synchronous replication)
```

### E2E Example 3: Post-Mortem Process

**Context:** Document incident for future prevention.

```markdown
# Post-Mortem: Payment Service Memory Leak
# Date: 2026-09-04
# Incident Commander: Rajesh Kumar

## Summary
Payment service experienced OutOfMemoryError during peak hours (salary day),
causing 15 minutes of degraded service affecting ~50,000 transactions.

## Timeline
- 10:00 AM: Alert received (error rate 12%)
- 10:02 AM: Incident commander joined
- 10:05 AM: Root cause identified (memory leak in TransferService)
- 10:08 AM: Mitigation applied (scaled up + increased memory)
- 10:12 AM: Service restored
- 10:30 AM: Monitoring confirmed stability

## Root Cause
Memory leak in TransferService.processTransfer() method.
Large batch transfers (10,000+ transactions) were not releasing
database connection objects, causing heap memory to grow over time.

## Impact
- Duration: 15 minutes
- Transactions affected: ~50,000 (0.02% of daily volume)
- Revenue impact: ₹0 (no transactions lost, just delayed)
- Customer complaints: 23 calls to support

## What Went Well
- Alert fired within 30 seconds
- Incident response team assembled in 2 minutes
- Mitigation applied in 6 minutes
- Service restored in 12 minutes

## What Could Be Improved
- Memory leak not caught in testing (need load testing with large batches)
- No circuit breaker for database connections
- Monitoring didn't show memory trend (need memory alerting)

## Action Items
1. [ ] Fix memory leak in TransferService (Owner: Rajesh, Due: 2026-09-11)
2. [ ] Add memory usage alerting (Owner: Priya, Due: 2026-09-18)
3. [ ] Implement load testing with 10,000+ transactions (Owner: Amit, Due: 2026-09-25)
4. [ ] Add circuit breaker for database connections (Owner: John, Due: 2026-09-25)

## Lessons Learned
- Always load test with production-scale data
- Monitor memory trends, not just current usage
- Circuit breakers prevent cascade failures
```

---

## 📋 Interview Questions

### Q1: What is the difference between incident response and problem management?
**Answer:** **Incident Management** focuses on restoring service quickly (reactive). **Problem Management** focuses on finding root cause and preventing recurrence (proactive). Incident: "Payment service is down — fix it now!" Problem: "Why did it go down? How do we prevent it?" Banks need both: fast incident response to minimize customer impact, thorough problem management to prevent future incidents.

### Q2: How do you write a good post-mortem?
**Answer:** Good post-mortems include: (1) **Blameless** — focus on systems, not people. (2) **Timeline** — minute-by-minute account. (3) **Root cause** — technical explanation, not just "human error." (4) **Impact** — quantify business impact. (5) **Action items** — specific, assigned, with deadlines. (6) **Lessons learned** — what to do differently. Bad post-mortem: "John made a mistake, be more careful." Good post-mortem: "Deploy pipeline didn't catch memory leak — add load testing."

### Q3: What is an incident commander and what do they do?
**Answer:** The Incident Commander (IC) leads the response effort. Responsibilities: (1) **Coordinate** — assemble the team, assign roles. (2) **Communicate** — status updates to stakeholders. (3) **Decide** — approve mitigation actions. (4) **Document** — record timeline and decisions. (5) **Escalate** — call in experts if needed. IC doesn't fix the issue — they manage the process. For P1 incidents, banks require dedicated IC who doesn't write code during the incident.

### Q4: How do you prevent alert fatigue in banking?
**Answer:** (1) **Symptom-based alerts** — alert on user impact (error rate > 1%), not causes (CPU > 80%). (2) **Severity levels** — P1: page immediately, P2: Slack, P3: email. (3) **Alert on trends** — sustained issues, not瞬时 spikes. (4) **Runbooks** — every alert has a documented response. (5) **Regular review** — remove alerts that never fire or never matter. (6) **Auto-resolve** — some alerts auto-close after conditions clear. Banks typically have 50-100 actionable alerts, not thousands.

### Q5: How do you measure incident response effectiveness?
**Answer:** Key metrics: (1) **MTTD** (Mean Time to Detect) — how fast alert fires (target: <1 minute). (2) **MTTA** (Mean Time to Acknowledge) — how fast team responds (target: <5 minutes for P1). (3) **MTTR** (Mean Time to Resolve) — how fast service is restored (target: <15 minutes for P1). (4) **Incident frequency** — how often incidents occur (target: decreasing). (5) **Action item completion** — are post-mortem actions completed on time? (6) **Customer impact** — transactions affected, complaints received.

---

## 📚 Summary

| Concept | Key Takeaway |
|---------|-------------|
| Incident Response | Detect → Triage → Mitigate → Resolve → Learn |
| Severity Levels | P1 (Critical) to P4 (Low) |
| Runbook | Step-by-step response procedures |
| Post-Mortem | Blameless, root cause, action items |
| MTTR | Mean Time to Resolve (target: <15min) |
| Banking Relevance | Zero data loss, minimal customer impact |

**Next:** [25-Banking-CICD-Interview-Mastery.md](./25-Banking-CICD-Interview-Mastery.md) — Final comprehensive interview guide.
