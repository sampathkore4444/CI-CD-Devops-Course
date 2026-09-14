# 103. Payment Service Deployed Successfully but 20% of Transactions Failing

## Scenario

At 10:00 AM, the deployment pipeline reports success — payment-service v2.5.0 has been deployed to production. All 6 pods are running, Kubernetes health checks (liveness and readiness probes) passed, the canary analysis showed no errors during the 15-minute observation window, and the deployment dashboard shows green across the board. The team celebrates and moves on to other tasks.

At 10:45 AM, alerts start firing from multiple monitoring systems. The payment failure rate has jumped to 20%. The failures are timeout errors — requests to the external payment gateway are timing out after 30 seconds. The failure rate is not uniform across the day: it spikes dramatically during peak business hours (10 AM - 12 PM, 2 PM - 4 PM) to 20-25%, drops to near-zero during off-peak hours (11 PM - 6 AM), and sits at about 8% during moderate traffic periods. The team deployed v2.5.0 which included three changes: connection pool configuration (reduced max connections from 50 to 20 per pod to save memory), timeout configuration (reduced from 30s to 10s based on "optimization"), and a new retry mechanism with exponential backoff. The full request path is: Mobile App → API Gateway → Payment Service (Kubernetes, 6 pods across 3 AZs) → Payment Switch → Core Banking → External Payment Network. You are the DevOps engineer on-call.

## Interviewer Question

"A payment service deployment passed all health checks but is now causing 20% of transactions to fail with timeout errors. The failures correlate with time of day — more failures during peak hours. Walk me through your investigation across the full stack and how you would resolve this. Consider connection pools, timeouts, rate limiting, DNS, TLS, load balancers, and circuit breakers."

## What I Should Think About

- Health check passing does not mean the service is functionally correct — health checks verify liveness, not business logic or capacity
- Time-of-day correlation is the critical clue — the issue is load-dependent, suggesting resource exhaustion under concurrency
- Connection pool sizing changes can cause timeouts under load but appear perfectly fine during low-traffic health checks
- Third-party payment gateway may have rate limits that are now being hit due to changed connection behavior
- Timeout configuration differences between versions can cause cascading timeouts — reducing timeout too aggressively causes premature failures
- DNS resolution caching can cause intermittent connection failures if TTL is misconfigured
- TLS certificate chain validation changes can cause silent failures that only appear under load
- Load balancer connection draining during deployment can cause brief failures that become persistent if draining is misconfigured
- Circuit breaker configuration may be too aggressive, causing cascade failures by rejecting requests prematurely
- Need to investigate the ENTIRE request path, not just the payment service — the issue could be anywhere in the chain
- The retry mechanism may be making things worse — retries add to connection pool pressure, creating a feedback loop
- A/B traffic splitting can help compare old vs new behavior in real-time

## Ideal Answer

This is a classic "works in low traffic, fails under load" scenario. The health checks passed because they don't generate real payment traffic through the external gateway — they typically just check if the HTTP port is open and a status endpoint returns 200. The time-of-day correlation is the critical diagnostic clue — the issue manifests when traffic volume increases, pointing to resource exhaustion.

**Most likely root cause:** The v2.5.0 connection pool configuration reduced the maximum connections from 50 to 20 per pod. Under low traffic (off-peak), 20 connections per pod is more than sufficient. During peak hours, 6 pods × 20 connections = 120 maximum concurrent connections to the payment gateway. The actual peak demand is approximately 150 concurrent connections. The excess 30+ requests queue in the connection pool, waiting for a connection to become available. Combined with the reduced timeout (30s → 10s), these queued requests timeout before a connection becomes available, resulting in the 20% failure rate.

**Secondary contributing factor:** The new retry mechanism retries failed requests, which adds to the connection pool pressure. If 20% of requests timeout, those 20% are retried, adding even more connection attempts to an already exhausted pool. This creates a vicious feedback loop that makes the failure rate worse during peak hours.

**Investigation approach:** Start from the client side and work inward through each layer of the stack. Check API Gateway error rates first, then payment service logs and connection pool metrics, then the payment switch, and finally the external gateway response times. The solution involves reverting the connection pool size, adjusting the timeout to match the gateway's actual response time profile, and disabling the retry mechanism temporarily.

## Investigation

1. Verify the deployment timeline: confirm v2.5.0 was deployed at 10:00 AM and failures started at 10:45 AM — correlate with the traffic increase during peak hours
2. Check API Gateway logs for the specific error types — are they 504 Gateway Timeout (gateway waiting for upstream) or 502 Bad Gateway (upstream connection failed)?
3. Review payment service logs for connection pool exhaustion warnings, "connection timeout" messages, or "pool exhausted" errors
4. Compare v2.4.0 vs v2.5.0 configuration files — specifically connection pool size (max connections), timeout values, and retry configuration
5. Check the payment switch logs to see if requests are arriving from the payment service but the gateway responses are slow
6. Query the external payment gateway API dashboard for their rate limits and your current usage — are you hitting a per-client connection limit?
7. Monitor connection pool metrics in real-time during peak vs off-peak hours: active connections, pending requests, wait time, connection creation rate
8. Check DNS resolution logs for any intermittent failures in resolving the payment gateway hostname — DNS caching TTL issues can cause periodic connection failures
9. Review TLS handshake logs — certificate chain validation changes can cause intermittent connection failures under load
10. Check load balancer connection draining settings — were connections from old pods properly drained during the deployment?

## Commands

```bash
# Compare v2.4.0 and v2.5.0 configuration side by side
diff <(kubectl get configmap payment-config-v240 -n production -o yaml) \
     <(kubectl get configmap payment-config-v250 -n production -o yaml)

# Check current connection pool metrics on a running pod
kubectl exec -it payment-service-xxx -n production -- \
  curl -s localhost:8080/actuator/metrics/hikaricp.connections.active | jq
kubectl exec -it payment-service-xxx -n production -- \
  curl -s localhost:8080/actuator/metrics/hikaricp.connections.pending | jq
kubectl exec -it payment-service-xxx -n production -- \
  curl -s localhost:8080/actuator/metrics/hikaricp.connections.timeout.total | jq

# View payment service logs for timeout and connection errors
kubectl logs -l app=payment-service -n production --tail=500 | \
  grep -i "timeout\|connection\|pool\|refused\|exhausted"

# Check the actual connection pool configuration in the running pod
kubectl exec -it payment-service-xxx -n production -- \
  env | grep -i "pool\|timeout\|connection\|retry"

# Monitor connection pool in real-time during peak hours
kubectl exec -it payment-service-xxx -n production -- watch -n 1 \
  "curl -s localhost:8080/actuator/metrics/hikaricp.connections.active | jq '.measurements[0].value'"

# Check API gateway error rates by upstream service
kubectl logs -l app=api-gateway -n production --tail=1000 | \
  jq 'select(.upstream == "payment-service")' | \
  jq -r '.status' | sort | uniq -c | sort -rn

# Check DNS resolution for payment gateway
kubectl exec -it payment-service-xxx -n production -- \
  nslookup payment-gateway.bank.com
kubectl exec -it payment-service-xxx -n production -- \
  dig payment-gateway.bank.com +short

# Check TLS certificate chain validity
kubectl exec -it payment-service-xxx -n production -- \
  openssl s_client -connect payment-gateway.bank.com:443 \
  -servername payment-gateway.bank.com </dev/null 2>/dev/null | \
  openssl x509 -noout -dates -issuer

# Check payment switch request/response timing
kubectl logs -l app=payment-switch -n production --tail=500 | \
  jq 'select(.service == "payment-service")' | \
  jq '{request_time: .request_time, response_time: .response_time, status: .status}'

# Monitor connection pool wait time via Prometheus
kubectl port-forward svc/prometheus 9090:9090 -n monitoring
# Query: hikaricp_connections_pending_seconds_max{job="payment-service"}

# Check load balancer connection draining status
aws elbv2 describe-target-health --target-group-arn <arn> | \
  jq '.TargetHealthDescriptions[] | {state: .TargetHealth.State, reason: .TargetHealth.Reason}'

# Compare timeout settings between versions
kubectl exec -it payment-service-xxx -n production -- \
  cat /app/application.yml | grep -A5 -B5 "timeout"

# Check retry count metrics
kubectl exec -it payment-service-xxx -n production -- \
  curl -s localhost:8080/actuator/metrics/payment.retry.count | jq

# Monitor database connection pool (payment service may also pool DB connections)
kubectl exec -it payment-service-xxx -n production -- \
  curl -s localhost:8080/actuator/metrics/hikaricp.connections.active | \
  jq '.measurements[0].value'
```

## Architecture

```
Full Request Path:
==================

┌──────────┐    ┌─────────┐    ┌──────────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Mobile   │───>│   API   │───>│   Payment    │───>│ Payment  │───>│   Core   │───>│ External │
│   App    │    │ Gateway │    │   Service    │    │  Switch  │    │ Banking  │    │ Gateway  │
└──────────┘    └─────────┘    └──────────────┘    └──────────┘    └──────────┘    └──────────┘
                       │              │                  │
                  Rate limit    Connection pool     Request routing
                  (100/s)       (20 per pod) ←BUG   (queue-based)

Connection Pool Exhaustion:
==========================

v2.4.0 (Working):
  Max Connections: 50/pod  |  Timeout: 30s  |  Pods: 6
  Total capacity: 300 concurrent connections
  Peak demand: ~150 connections  →  50% utilization  ✓

v2.5.0 (Failing):
  Max Connections: 20/pod  |  Timeout: 10s  |  Pods: 6
  Total capacity: 120 concurrent connections
  Peak demand: ~150 connections  →  125% utilization  ✗

  Excess requests queue in pool → wait → timeout at 10s → FAILURE
  Retries add more requests → feedback loop → worse failures

Time-of-Day Failure Pattern:
============================

Failure % 
  25% │          ████              ████
  20% │        ████████          ████████
  15% │      ████████████      ████████████
  10% │    ████████████████  ████████████████
   5% │  ████████████████████████████████████
   0% │████████████████████████████████████████
      └──6AM──9AM──12PM──3PM──6PM──9PM──12AM──
           Peak: 10AM-12PM, 2PM-4PM
           Off-peak: 11PM-6AM (near-zero failures)

Retry Feedback Loop:
====================
1. Request arrives → pool exhausted → waits
2. Wait exceeds timeout (10s) → request fails
3. Retry mechanism retries the failed request
4. Retry also hits pool → also exhausted → also fails
5. Another retry → even more pressure → cascade
```

Connection Pool Exhaustion Detail:
==================================

Normal (v2.4.0):                      Degraded (v2.5.0):
                                      
  Pod 1: [==C==......] 2/50           Pod 1: [CCCCCCCCCCCCCCCCCCCC] 20/20
  Pod 2: [==C==......] 2/50           Pod 2: [CCCCCCCCCCCCCCCCCCCC] 20/20
  Pod 3: [==C==......] 2/50           Pod 3: [CCCCCCCCCCCCCCCCCCCC] 20/20
  Pod 4: [==C==......] 2/50           Pod 4: [CCCCCCCCCCCCCCCCCCCC] 20/20
  Pod 5: [==C==......] 2/50           Pod 5: [CCCCCCCCCCCCCCCCCCCC] 20/20
  Pod 6: [==C==......] 2/50           Pod 6: [CCCCCCCCCCCCCCCCCCCC] 20/20
                                      
  Total: 12/300 (4% utilized)         Total: 120/120 (100% EXHAUSTED)
                                      Pending: 30 requests WAITING...
                                      Timeout at 10s → 30 requests FAIL

Retry Feedback Loop:
====================

  Normal requests ──> [Connection Pool] ──> Payment Gateway
                          │ (FULL!)
                          │
                     Timeout at 10s
                          │
                     Retry request ──> [Connection Pool] ──> STILL FULL!
                          │
                     Another timeout
                          │
                     Another retry ──> [Connection Pool] ──> CASCADING FAILURE

  This creates a vicious cycle during peak hours.

Timeout Configuration Impact:
=============================

v2.4.0: Timeout = 30s
  - Requests wait up to 30s for a connection
  - Most requests get a connection within 5s
  - Very few timeouts even during peak

v2.5.0: Timeout = 10s
  - Requests wait only 10s for a connection
  - During peak, pool is full → requests timeout at 10s
  - Retries add more pressure → more timeouts
  - 20% failure rate during peak hours

Health Check vs Real Traffic:
=============================

Health Check:                          Real Traffic:
  GET /health → 200 OK                  POST /payment → timeout
  (no gateway call)                     (full stack exercise)
  (1 request/30s)                       (100s of requests/sec)
  (always passes)                       (20% fail during peak)
```

## Root Cause

**Primary:** v2.5.0 reduced the connection pool maximum from 50 to 20 per pod. During peak hours, 6 pods × 20 = 120 maximum concurrent connections to the payment gateway. The actual peak demand is approximately 150 concurrent connections. The excess 30+ requests queue in the connection pool, and the reduced timeout (30s → 10s) causes them to fail before a connection becomes available.

**Secondary:** The new retry mechanism retries failed requests, which adds to the connection pool pressure, creating a feedback loop that amplifies the failure rate during peak hours.

**Contributing:** The health check endpoint does not exercise the payment gateway connection, so it passed during deployment validation. The canary analysis only ran for 15 minutes, which may not have overlapped with peak traffic.

## Immediate Mitigation

1. Revert the connection pool size to 50 per pod in the configuration immediately
2. Revert the timeout to 30s or set it to a value appropriate for the payment gateway's actual response time (measure p99 first, then set timeout to 2x p99)
3. Disable the retry mechanism temporarily to break the feedback loop
4. Redeploy with the reverted configuration using a rolling deployment
5. Monitor failure rate in real-time to confirm resolution within 15 minutes

## Permanent Fix

1. Load test the payment service with realistic peak traffic before deployment to catch connection pool sizing issues — this should be a deployment gate
2. Implement connection pool monitoring with alerts for pool exhaustion (pending connections > 0)
3. Set the connection pool maximum based on measured peak concurrent connections + 30% headroom
4. Configure timeout based on payment gateway's p99 response time, not an arbitrary value — measure first, then set
5. Implement adaptive timeout based on historical latency percentiles (e.g., timeout = p99 × 2)
6. Add canary analysis that specifically tests payment processing during peak traffic hours — not just any 15-minute window
7. Create a deployment checklist that includes connection pool validation against production traffic patterns
8. Implement circuit breaker for payment gateway calls — if connection pool is exhausted, fail fast instead of queuing

## Monitoring

- **Connection pool metrics:** Active connections, pending requests, wait time, connection creation rate, pool utilization percentage
- **Timeout metrics:** Requests timing out at each layer (API Gateway, payment service, payment switch) — track separately
- **Payment gateway response time:** p50, p95, p99 latency measured from the payment service to the external gateway
- **Failure rate by time of day:** Track failure rate hourly to detect load-correlated issues early
- **Connection pool exhaustion events:** Alert immediately when pending connections > 0 or pool utilization > 80%
- **Retry rate:** Monitor retry frequency — high retry rates indicate upstream instability and contribute to feedback loops
- **Thread pool metrics:** Monitor Tomcat/Jetty thread pool — connection pool exhaustion often leads to thread pool exhaustion

## Security

- Connection pool exhaustion can be exploited as a denial-of-service vector — implement per-client rate limiting at the API gateway
- TLS certificate validation must not be weakened (e.g., disabling certificate verification) to "fix" connection issues
- Payment gateway credentials must be rotated regularly regardless of connection pool changes
- Connection pool monitoring should not expose sensitive payment data in metrics or logs
- Timeout values must not be set so high that they allow slow-loris style attacks against the payment service
- Rate limiting at the API gateway must account for connection pool capacity — incoming rate must not exceed gateway connection capacity

## Production Considerations

- **High Availability:** Connection pool exhaustion on one pod can cascade — the load balancer must detect and drain unhealthy pods that are not responding due to pool exhaustion
- **Scalability:** Connection pool sizing must be reviewed when pods are autoscaled — more pods means more total connections, which may exceed the payment gateway's per-client connection limit
- **Cost:** Larger connection pools consume more memory, but the cost of failed transactions far exceeds memory cost — a failed $500 transaction costs more in customer trust and remediation than 1GB of additional memory
- **Third-party Limits:** Payment gateways have connection and rate limits — these must be documented, monitored, and enforced at the API gateway layer
- **Load Testing:** Connection pool issues only appear under realistic load — health checks are fundamentally insufficient for capacity validation
- **Observability:** Connection pool metrics must be visible in the deployment dashboard during canary analysis, not buried in application logs

## Senior-Level Answer

"The 20% failure rate correlating with time of day points to a connection pool exhaustion issue under peak load. Health checks passed because they don't exercise the payment gateway — they only verify the HTTP port is open. I would compare the v2.5.0 connection pool configuration with v2.4.0, check connection pool utilization metrics during peak vs off-peak, and verify the payment gateway's rate limits. The fix is to size the connection pool based on measured peak concurrent connections plus 30% headroom, set timeouts based on the gateway's p99 latency multiplied by 2, and add connection pool exhaustion alerts. For future deployments, I would add load testing as a deployment gate for payment-critical services."

## Architect-Level Answer

"This incident exposes a gap in our deployment validation pipeline. Health checks verify liveness, not capacity. We need to implement deployment gates that include load testing against production-like traffic patterns before promoting a canary. The connection pool is a shared resource that affects the entire request path — we need capacity planning that accounts for the full stack, not just individual service metrics. I recommend implementing adaptive connection pooling that adjusts pool size based on real-time metrics, circuit breakers that shed load before pool exhaustion causes cascading failures, and timeout values that are dynamically calculated from historical latency percentiles. The third-party payment gateway rate limits should be documented as infrastructure constraints in our service level agreements and enforced at the API gateway layer — not discovered in production. Additionally, we need to implement traffic-aware canary analysis that specifically tests during peak hours, not just any random 15-minute window."

## Follow-Up Questions

1. "How would you implement a deployment gate that catches connection pool issues before they reach production?"
2. "What is the difference between connect timeout and read timeout, and why does it matter for payment gateway integrations?"
3. "How would you handle this scenario if the payment gateway itself had a hard rate limit and you couldn't increase your connection pool?"
4. "How would you implement an adaptive timeout that adjusts based on the payment gateway's actual response time percentiles?"
5. "If you needed to deploy the v2.5.0 changes (which include a security fix) but can't revert the connection pool, how would you safely deploy?"

## A/B Traffic Splitting for Version Comparison

When investigating deployment-related failures, A/B traffic splitting can help compare old vs new behavior in real-time:

**Implementation:** Use Istio VirtualService or NGINX canary annotations to split traffic 50/50 between v2.4.0 (stable) and v2.5.0 (canary). This allows direct comparison of behavior under identical traffic conditions. The VirtualService configuration below routes 50% of requests to each version:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
  - payment-service
  http:
  - route:
    - destination:
        host: payment-service
        subset: v2-4-0
      weight: 50
    - destination:
        host: payment-service
        subset: v2-5-0
      weight: 50
```

**Metrics Comparison:** Compare the following metrics between versions in real-time: payment success rate, average latency, connection pool utilization, error rate by type, and retry rate. If v2.5.0 shows worse metrics, the root cause is in the new version. Use a dashboard that splits metrics by version label.

**Gradual Shift:** Start with 90/10 (90% stable, 10% canary) and monitor. If canary metrics are acceptable, increase to 70/30, then 50/50. If metrics degrade, shift back to 100/0 and investigate.

**Benefits for This Scenario:** A/B splitting allows the team to keep the security patch deployed (to a small percentage of users) while investigating the connection pool issue. This balances security requirements with stability. Once the connection pool is fixed, the patch can be fully promoted.

## Post-Incident Review Checklist

After the incident is resolved, conduct a thorough review:

**1. Timeline Reconstruction:** Document the exact timeline: deployment start, deployment complete, first alert, investigation start, fix deployed, failure rate back to normal. This timeline becomes the template for future incident reviews.

**2. Root Cause Confirmation:** Verify the diagnosed root cause matches the evidence. Check: connection pool utilization at failure time, timeout counts in logs, gateway response times. Do not accept a hypothesis without evidence.

**3. Blameless Postmortem:** Document contributing factors without blaming individuals. Focus on systemic gaps: why did health checks not catch this? Why was the connection pool changed without load testing? What process allowed this config change through?

**4. Action Items:** Create and track action items in priority order:
- P0: Add load testing as a deployment gate (week 1)
- P0: Add connection pool alarms (week 1)
- P1: Implement adaptive timeout based on latency percentiles (month 1)
- P1: Add peak-hour canary analysis (month 1)
- P2: Implement circuit breaker for payment gateway (month 2)

**5. Configuration Change Control:** Implement mandatory peer review for all production configuration changes. The connection pool change from 50 to 20 should have required review and load testing. Configuration changes need the same rigor as code changes.

**6. Runbook Update:** Update the deployment and incident runbooks with the lessons learned from this incident. Ensure the runbooks include the specific diagnostic commands and queries used during this investigation.

## Canary Analysis Configuration for Payment Services

Canary analysis must include business-specific metrics, not just technical metrics:

**Payment Success Rate:** The percentage of payment requests that complete successfully. Must remain above 99.9% during canary. A drop from 99.95% to 99.80% may seem small but represents 15 additional failed transactions per 10,000 — each potentially a customer-impacting failure.

**Gateway Response Latency:** The p99 latency from the payment service to the external gateway. If the canary shows higher latency than stable, it may indicate connection issues. Set threshold: canary p99 must not exceed stable p99 by more than 20%.

**Connection Pool Utilization:** Monitor the connection pool utilization on canary pods. If canary pods show higher pool utilization than stable, the new version may have a connection leak or sizing issue.

**Error Classification:** Not all errors are equal. A 502 (Bad Gateway) is more concerning than a 504 (Gateway Timeout). Track error types separately and set different thresholds for each.

**Time-Based Analysis:** Run canary analysis during peak hours, not just during deployment. If deployment happens at 10 AM, extend the canary observation window to cover the 10 AM-12 PM peak period.

## DNS and TLS Investigation Considerations

While connection pool exhaustion is the most likely root cause, other network-layer issues can produce similar time-of-day failure patterns:

**DNS Resolution Caching:** If the payment gateway's DNS record has a short TTL (e.g., 60 seconds) and the DNS server is slow during peak hours, DNS resolution delays can cause connection timeouts. Check DNS resolution time: `dig payment-gateway.bank.com | grep "Query time"`.

**TLS Certificate Chain:** If the v2.5.0 changed TLS configuration or updated the Java version, certificate chain validation might fail intermittently. Check: `openssl s_client -connect payment-gateway.bank.com:443 </dev/null 2>/dev/null | openssl x509 -noout -text`.

**Load Balancer Connection Draining:** During deployment, old pods are drained from the load balancer. If draining is not configured properly, in-flight requests may be terminated prematurely, causing failures that persist after deployment.

**Network Congestion:** Peak hours may coincide with network congestion on shared infrastructure. Check network latency to the payment gateway: `ping payment-gateway.bank.com` and `traceroute payment-gateway.bank.com`.

## Load Testing as a Deployment Gate

The v2.5.0 incident could have been prevented with load testing as a deployment gate:

**Pre-Deployment Load Test:** Before promoting a canary to production, run a load test against the staging environment with production-equivalent traffic patterns. This should be automated in the CI/CD pipeline.

**Traffic Replay:** Capture 15 minutes of production traffic and replay it against the staging environment with the new version. Compare metrics: latency, error rate, connection pool utilization, memory usage.

**Peak Hour Simulation:** The load test must simulate peak hour traffic, not average traffic. The v2.5.0 connection pool was sized for average traffic (50 connections sufficient) but failed during peak (150 concurrent connections needed).

**Connection Pool Monitoring During Test:** During the load test, monitor connection pool metrics in real-time. If pending connections > 0 at any point during the test, the deployment should be blocked.

**Automated Gate:** Implement the load test as a pipeline stage that must pass before canary promotion. If the test fails, the deployment is blocked and the team is notified.

## Connection Pool Best Practices for Financial Services

Connection pool configuration is critical for payment services. Here are the key principles:

**Sizing Formula:** Pool size = (peak concurrent requests × average response time in seconds) + headroom. For this scenario: 150 peak concurrent × 0.5s average = 75 minimum. Add 30% headroom = 98. The v2.5.0 setting of 120 total (20×6) was insufficient for the 150 peak demand.

**Timeout Configuration:** Set connect timeout to 5s (TCP handshake). Set read timeout to 2× p99 latency of the downstream service. If the payment gateway's p99 is 300ms, set read timeout to 600ms. The v2.5.0 setting of 10s was too aggressive — it caused premature failures before connections became available.

**Monitoring Alerts:** Alert when pool utilization exceeds 70% (early warning), 85% (elevated), or 95% (critical). Track pending connections — any non-zero value means requests are queuing.

**Connection Validation:** Enable connection validation (test query on borrow) to prevent using stale connections. For payment gateways, validate that the connection can reach the gateway before sending a payment request.

**Warm-up:** After deployment, warm up the connection pool by sending a small number of requests before handling production traffic. This prevents cold-start failures where all requests contend for new connections simultaneously.
