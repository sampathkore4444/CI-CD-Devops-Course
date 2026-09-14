# 104. Core Banking System Down - Mobile Banking Operational

## Scenario

At 9:30 AM on a Monday morning, the core banking system (CBS) goes completely down due to a storage array failure at the primary data center. The SAN controller lost redundancy and failed over to a non-responsive secondary, causing the entire storage subsystem to become unreachable. The CBS team estimates a 4-hour recovery window — they need to restore the SAN from backup, verify data integrity, and bring the database back online.

Meanwhile, the mobile banking application continues to run — users can log in (session service is separate from CBS), view the home screen, navigate the app, and access cached data. However, all transactions that require real-time CBS interaction (transfers, payments, bill pay, balance inquiries that hit CBS) are failing with 503 Service Unavailable errors. The support center is receiving 200+ calls per minute. Customers are panicking, especially business customers with time-sensitive payments due today. Social media is starting to pick up complaints. You are the DevOps architect responsible for designing and implementing the mobile banking application's behavior during this outage. The mobile app runs on Kubernetes with Java/Spring Boot microservices.

## Interviewer Question

"The core banking system is completely down for an estimated 4 hours. Mobile banking is still running but transactions requiring CBS are failing. Design the complete response: how the application should behave during the outage, how to protect customers, how to queue transactions for later processing, and how to reconcile after recovery. Address circuit breaker, timeout, retry, fallback, queue, reconciliation, and customer notification."

## What I Should Think About

- Circuit breaker pattern must prevent cascading failures from repeated CBS connection attempts — without it, every request blocks a thread for the full timeout duration
- Timeout configuration must be short enough to prevent thread pool exhaustion but long enough for normal operations
- Retry with exponential backoff prevents overwhelming a struggling CBS during partial recovery attempts
- Fallback strategies: cached balance with staleness warning, queue transaction for later processing, serve cached statements
- Message queue for buffering transactions during outage preserves customer intent without data loss
- Reconciliation after recovery must prevent duplicate processing and ensure transaction ordering
- Customer communication must be proactive, not reactive — status page, in-app messaging, push notifications
- Graceful degradation: which features work without CBS (profile, settings, cached history), which don't (transfers, real-time balance)
- Compliance: financial transaction audit trail must be maintained even during degraded mode — queued transactions need audit entries
- CBS recovery: replaying queued transactions without creating duplicates requires idempotency keys
- Business customers with time-sensitive payments need priority handling in the queue

## Ideal Answer

**Layer 1 — Circuit Breaker:** Immediately stop sending requests to CBS after detecting the failure. The circuit breaker opens after 5 consecutive failures (within 10 seconds), preventing thread pool exhaustion. When CBS starts recovering, the circuit breaker enters half-open state and tests with a single request before fully closing. This protects the mobile banking application from being brought down by CBS unavailability — without circuit breaker, 30-second timeouts × thousands of requests = thread pool exhaustion in minutes.

**Layer 2 — Fallback Responses:**
- Balance inquiry: Return the last cached balance from Redis with a prominent "Balance as of [timestamp] — may not reflect recent transactions" warning
- Transfer/Payment: Accept the transaction intent into a durable queue, return a reference number to the customer with "Your transfer has been saved and will be processed when services are restored"
- Statement: Serve the last synced statement from the cache
- Profile/Settings: These work normally — they don't require CBS

**Layer 3 — Transaction Queue:** Accepted transactions are written to a durable message queue (Kafka or RabbitMQ) with the customer's complete intent. Each queued transaction includes: customer ID, amount, recipient account, idempotency key, timestamp, priority level (business vs personal), and expiration time. The queue must be durable — if the queue itself fails, customer transactions are lost, which is unacceptable.

**Layer 4 — Customer Communication:** Display an in-app banner: "Some services are temporarily unavailable. Your transactions are saved and will be processed shortly." Push notification to customers with pending transactions. Update the status page with estimated restoration time.

**Layer 5 — Reconciliation:** After CBS recovery, replay queued transactions in priority order (business customers and time-sensitive payments first). Verify each transaction was processed exactly once using idempotency keys. Notify customers of completion via push notification. Log all reconciliation results for audit compliance.

## Investigation

1. Confirm CBS is completely down — check all health endpoints, not just the primary, and verify at the network level
2. Determine the CBS team's estimated recovery time and get regular updates
3. Identify which mobile banking features require CBS (transfers, balance, payments) vs which don't (profile, settings, cached history)
4. Check if the circuit breaker is already tripped — if not, manually activate it to stop CBS connection attempts immediately
5. Verify the message queue is operational and can handle the expected transaction volume during the 4-hour outage
6. Check cached balance freshness — when was the last sync with CBS? How stale is the cached data?
7. Assess support center load — how many customers are affected and how quickly are calls coming in?
8. Verify the status page is operational and can display the outage information to customers
9. Check if any batch jobs or scheduled transactions will attempt to hit CBS during the outage and must be paused
10. Confirm the reconciliation process is ready to handle queued transactions after CBS recovery

## Commands

```bash
# Check circuit breaker state
kubectl exec -it mobile-banking-api-xxx -n production -- \
  curl -s localhost:8080/actuator/circuitbreakers | jq

# Force circuit breaker open if not already
kubectl exec -it mobile-banking-api-xxx -n production -- \
  curl -X POST localhost:8080/actuator/circuitbreakers/cbs-connection/open

# Check circuit breaker configuration
kubectl exec -it mobile-banking-api-xxx -n production -- \
  curl -s localhost:8080/actuator/circuitbreakers/cbs-connection | jq

# Check message queue depth (queued transactions)
kubectl exec -it rabbitmq-0 -n production -- \
  rabbitmqctl list_queues name messages consumers | grep transaction

# Check cached balance freshness
kubectl exec -it redis-cache-0 -n production -- \
  redis-cli GET "balance:last_sync_timestamp"

# Monitor circuit breaker metrics in real-time
kubectl exec -it mobile-banking-api-xxx -n production -- \
  curl -s localhost:8080/actuator/metrics/resilience4j.circuitbreaker.state | jq

# Check thread pool for CBS connections (prevent exhaustion)
kubectl exec -it mobile-banking-api-xxx -n production -- \
  curl -s localhost:8080/actuator/metrics/tomcat.threads.busy | jq

# View queued transactions count
kubectl exec -it rabbitmq-0 -n production -- \
  rabbitmqctl list_queues name messages | grep transaction

# Check in-flight transactions that might be stuck
kubectl logs -l app=mobile-banking-api -n production --tail=200 | \
  grep "CBS_TIMEOUT\|circuit_breaker\|FALLBACK\|QUEUE_ACCEPTED"

# Verify CBS recovery
kubectl logs -l app=core-banking -n production --tail=50 | grep "SYSTEM_READY"

# Replay queued transactions after CBS recovery (dry run first)
kubectl exec -it transaction-replay-processor -n production -- \
  java -jar replay.jar --priority=time-sensitive --dry-run

# Actually replay after verification
kubectl exec -it transaction-replay-processor -n production -- \
  java -jar replay.jar --priority=all --batch-size=100

# Check reconciliation status
kubectl exec -it reconciliation-service -n production -- \
  curl -s localhost:8080/api/reconciliation/status | jq

# Pause scheduled batch jobs during outage
kubectl patch cronjob nightly-settlement -n production -p \
  '{"spec":{"suspend":true}}'

# Update status page
kubectl exec -it status-page-deployer -n production -- \
  java -jar statuspage.jar --component=mobile-banking --status=degraded \
  --message="Transfers temporarily unavailable. Transactions saved for processing."

# Send push notification to affected customers
kubectl exec -it notification-service -n production -- \
  curl -X POST localhost:8080/api/notify/transaction-queued \
  -d '{"customer_ids": [...], "message": "Your transaction has been saved"}'
```

## Architecture

```
During CBS Outage:
==================

┌──────────┐    ┌─────────┐    ┌──────────────┐    ┌──────────────┐
│ Mobile   │───>│   API   │───>│   Mobile     │───>│ Core Banking │
│   App    │    │ Gateway │    │ Banking API  │    │    (DOWN)    │
└──────────┘    └─────────┘    └──────┬───────┘    └──────────────┘
                                      │
                              ┌───────┴────────┐
                              │ Circuit Breaker │
                              │   STATE: OPEN   │
                              │ (5 failures →   │
                              │  stop sending)  │
                              └───────┬────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    │                 │                  │
             ┌──────┴──────┐  ┌──────┴──────┐  ┌───────┴──────┐
             │  BALANCE     │  │  TRANSACTION │  │   CACHED     │
             │  FALLBACK    │  │    QUEUE     │  │   RESPONSE   │
             │ (cached +   │  │ (Kafka/RMQ) │  │  (statements,│
             │  staleness   │  │ durable,     │  │   history)   │
             │  warning)    │  │ prioritized  │  │              │
             └─────────────┘  └──────┬───────┘  └──────────────┘
                                     │
                              ┌──────┴───────┐
                              │  Reconciler   │
                              │ (after CBS    │
                              │  recovery)    │
                              │ idempotent    │
                              │ replay        │
                              └──────────────┘

Circuit Breaker State Machine:
==============================

CLOSED ──(5 failures/10s)──> OPEN ──(30s timer)──> HALF-OPEN
  ^                                                  │
  │                                        (success)  │ (failure)
  └──────────────────────────────────────────────────┘
                                                       │
                                                  OPEN (retry timer)

Customer Experience During Outage:
==================================
┌─────────────────────────────────────────────┐
│  Mobile Banking App                          │
│                                              │
│  ⚠️ Some services temporarily unavailable    │
│  Core banking maintenance in progress.       │
│  Your transactions are saved and will be     │
│  processed shortly.                          │
│                                              │
│  Balance: $12,500.00                         │
│  (as of 9:28 AM — before maintenance)        │
│                                              │
│  [Transfer] → Accepts → "Ref# TXN-2024..."  │
│  [Pay Bill] → Accepts → "Ref# TXN-2024..."  │
│  [History]  → Shows from cache               │
│  [Settings] → Works normally                 │
│                                              │
│  ──────────────────────────────────────────  │
│  Pending Transactions: 1                     │
│  • $500 to Vendor ABC — Queued, Ref# TXN... │
│                                              │
└─────────────────────────────────────────────┘

Reconciliation After Recovery:
==============================

1. CBS reports SYSTEM_READY
2. Circuit breaker enters HALF-OPEN → tests single request
3. Test request succeeds → circuit breaker CLOSES
4. Reconciliation processor starts
5. Reads queue in priority order:
   a. Business/time-sensitive transactions first
   b. Personal transactions second
   c. For each: check idempotency key in CBS
   d. If not processed: submit transaction
   e. If already processed: skip (dedup)
6. Notify customer of completion
7. Log all reconciliation results for audit
```

## Root Cause

**Primary:** CBS storage array failure — a hardware issue at the primary data center. The SAN controller lost redundancy, the failover to the secondary controller failed, and the entire storage subsystem became unreachable. This is an infrastructure failure, not an application bug.

**Application Gap:** The mobile banking application was not designed for graceful degradation. Without circuit breaker and fallback patterns, every request to CBS would block a thread for the full timeout duration (30 seconds), eventually exhausting the thread pool and causing the entire mobile banking application to become unresponsive — even for features that don't need CBS (profile, settings, cached history).

**Missing Capabilities:**
- No circuit breaker to stop sending requests to a failed CBS
- No fallback for balance inquiries (last cached balance)
- No transaction queue to preserve customer intent during outage
- No automated customer communication during degradation
- No reconciliation process for queued transactions after recovery
- No priority handling for business-critical transactions

## Immediate Mitigation

1. Manually activate circuit breaker to stop CBS connection attempts — this prevents thread pool exhaustion
2. Deploy a hotfix that returns cached balance with staleness warning (if not already implemented)
3. Implement transaction queue for accepted transfers/payments with idempotency keys
4. Display in-app banner explaining the situation and estimated restoration time
5. Update status page with outage information
6. Redirect support center calls to automated message with status update
7. Disable any batch jobs or scheduled transactions that would attempt CBS connections
8. Alert the compliance team about the degraded mode operation

## Permanent Fix

1. Implement Resilience4j circuit breaker with proper configuration (5 failures, 30s wait, half-open test)
2. Implement transaction outbox pattern — write intent to local database transactionally, then asynchronously process via message queue
3. Build a reconciliation service that replays queued transactions after CBS recovery using idempotency keys
4. Implement cached balance service with configurable staleness threshold and prominent warnings
5. Create automated customer notification system for service degradation events
6. Build a degradation dashboard showing which features are degraded and why
7. Implement priority-based transaction queue (business customers, time-sensitive payments first)
8. Conduct quarterly outage simulation drills to test the entire degradation and recovery flow
9. Implement real-time CBS health monitoring with automatic circuit breaker activation

## Monitoring

- **Circuit breaker state:** Alert on every state change (closed → open → half-open → closed)
- **CBS connection success rate:** Monitor in real-time during partial recovery to detect flapping
- **Transaction queue depth:** Alert if queue exceeds 10,000 transactions during outage — indicates extended outage
- **Cached balance staleness:** Alert if cache is older than 5 minutes — stale data must be clearly marked
- **Customer impact:** Track number of degraded transactions and affected customers for regulatory reporting
- **Recovery metrics:** Track CBS recovery time, queue drain rate, reconciliation accuracy, duplicate detection rate
- **Support center volume:** Monitor call/chat volume correlation with outage for capacity planning

## Security

- Queued transactions must be encrypted at rest in the message queue — financial data at rest must be protected
- Cached balances must not be accessible to unauthorized services — implement proper access controls
- Reconciliation process must verify transaction integrity — no duplicate processing, no lost transactions
- Customer notification must not expose sensitive transaction details in push notifications (amounts, recipients)
- Circuit breaker must not be bypassed by any service — it is a safety mechanism, not a suggestion
- Queued transactions must maintain the customer's original authentication context for replay — don't lose the authorization
- Degraded mode must still enforce transaction limits and fraud checks

## Production Considerations

- **High Availability:** The circuit breaker, transaction queue, and cache must all be highly available — they become the critical path during outage. A single point of failure in any of these turns a CBS outage into a complete mobile banking outage.
- **Scalability:** The transaction queue must handle peak outage volume — potentially 10x normal transaction rate as customers retry. Size the queue for worst-case scenario.
- **Data Integrity:** Queued transactions must be durable — if the queue itself fails during outage, customer transactions are lost. Use persistent volumes and replication.
- **Compliance:** Financial regulators require audit trails for all transactions, including those queued during outage. The queue must maintain full audit metadata.
- **Cost:** Running a message queue and cache adds infrastructure cost, but the cost of lost transactions and customer trust is far higher.
- **Recovery:** CBS recovery must be tested regularly — a 4-hour recovery is only useful if the queue can actually replay that many transactions within a reasonable time after recovery.

## Senior-Level Answer

"I would implement a four-layer defense: circuit breaker to stop CBS connection attempts and prevent thread pool exhaustion, cached balance fallback for balance inquiries with staleness warnings, transaction queue for transfers/payments with idempotency keys, and customer communication via in-app banners and status page. The circuit breaker opens after 5 consecutive CBS failures. Transactions are accepted into a durable queue with priority ordering (business customers first). After CBS recovery, the reconciliation service replays queued transactions using idempotency keys to prevent duplicates. This design ensures mobile banking remains functional during CBS outages, customers are informed, and no transactions are lost."

## Architect-Level Answer

"This scenario requires designing for degradation as a first-class capability, not an afterthought. The architecture should implement the outbox pattern at the data layer — every transaction intent is written to a local database transactionally, then published to a message queue asynchronously. During CBS outage, the local database accepts transactions, and a background processor queues them for later. The reconciliation service must handle idempotency, ordering, and partial failures during replay. We need a degradation taxonomy: Tier-1 services must always work (authentication, caching), Tier-2 services degrade gracefully (balance with staleness, queued transactions), Tier-3 services can fail (real-time notifications, statement generation). CBS connectivity is Tier-2 — the mobile app must function without it. The customer communication layer should be automated and integrated with our status page system. Finally, we need to test this design with quarterly chaos engineering drills that simulate CBS outage and measure recovery time, transaction preservation rate, and customer impact. The goal is: during any single-system outage, the mobile banking experience degrades gracefully rather than fails catastrophically."

## Summary: Degraded Mode Feature Matrix

```
Feature            │ CBS Required? │ During Outage        │ Customer Experience
───────────────────┼───────────────┼──────────────────────┼────────────────────
Login/Logout       │ No            │ WORKS normally       │ Normal
View Profile       │ No            │ WORKS normally       │ Normal
View Settings      │ No            │ WORKS normally       │ Normal
Balance Inquiry    │ Yes           │ CACHED (stale OK)    │ "Balance as of 9:28 AM"
Transfer           │ Yes           │ QUEUED               │ "Ref# TXN-2024 saved"
Bill Payment       │ Yes           │ QUEUED               │ "Ref# TXN-2024 saved"
Payment            │ Yes           │ QUEUED               │ "Ref# TXN-2024 saved"
View History       │ Yes           │ CACHED (may be stale)│ "Showing cached data"
Statement          │ Yes           │ CACHED               │ "Last synced: 9:28 AM"
Push Notifications │ No            │ DEGRADED             │ Delayed delivery
```

## Follow-Up Questions

1. "How do you ensure the message queue itself is highly available — what happens if RabbitMQ/Kafka goes down during a CBS outage?"
2. "How would you handle a scenario where CBS partially recovers — some transactions work, some fail — without losing or duplicating queued transactions?"
3. "What is the maximum acceptable queue depth before you start rejecting new transactions, and how do you decide that number?"
4. "How do you handle priority ordering in the queue — should a CEO's $1M wire transfer be processed before a retail customer's $50 transfer?"
5. "How would you implement a 'transaction status' feature in the mobile app so customers can see the real-time status of their queued transactions?"

## Post-Recovery Verification Checklist

After CBS recovery and transaction replay, verify system health comprehensively:

**1. CBS Health:** Verify all CBS components are healthy — database, storage array, SAN controller, network connectivity. Check replication status if using read replicas.

**2. Transaction Queue Drain:** Confirm all queued transactions have been processed or moved to dead letter queue. Queue depth should be zero (or only contain failed transactions for manual review).

**3. Reconciliation Results:** Run reconciliation between payment service records and CBS records. All queued transactions should have a matching CBS entry. No duplicates should exist.

**4. Customer Notifications:** Confirm all customers with queued transactions have received completion notifications. Check notification delivery logs.

**5. System Metrics:** Verify all system metrics have returned to normal — database CPU, connection pool utilization, cache hit rate, response latency, error rate.

**6. Support Center Volume:** Monitor support center call volume — it should decrease as customers receive completion notifications.

**7. Audit Trail:** Verify that all transactions during the outage window — both queued and processed — have complete audit trail entries.

**8. Scheduled Jobs:** Resume any batch jobs or scheduled transactions that were paused during the outage.

## Customer Communication During Outage

Proactive communication during a banking outage is critical for maintaining customer trust:

**In-App Messaging:** Display a prominent banner at the top of the mobile banking app explaining the situation. Include: what is affected, what customers can do, and the estimated restoration time. Update the banner as the situation evolves.

**Status Page:** Maintain a public status page (status.yourbank.com) that shows the status of each service. Customers and support staff can check this page for real-time updates without calling the support center.

**Push Notifications:** Send push notifications to customers with queued transactions confirming that their transaction has been saved and will be processed. Include a reference number they can use to track the status.

**Social Media Response:** Monitor social media for customer complaints. Respond promptly with a message directing customers to the status page and assuring them that transactions are being preserved.

**Support Center Scripts:** Provide support staff with scripts that explain the outage, what is being done, and what customers can expect. Avoid technical jargon — focus on what customers care about: their money is safe, their transactions are saved, and service will be restored soon.

## Testing the Degradation Design

The degradation design must be tested regularly to ensure it works when needed:

**Quarterly Outage Drills:** Simulate a CBS outage by shutting down the CBS connection at the network level. Measure: time to detect, time to activate circuit breaker, time to start queuing transactions, customer impact, and recovery time after CBS is restored.

**Chaos Engineering:** Use tools like Chaos Monkey or Litmus to randomly kill Redis pods, network connections, or CBS endpoints. Observe how the application behaves and whether degradation is graceful.

**Load Testing Under Degraded Mode:** Run load tests with the circuit breaker open to verify the application can handle peak traffic during an outage without the database crashing.

**Recovery Testing:** After simulating an outage, restore the connection and verify that queued transactions are replayed correctly, no duplicates are created, and all customers are notified.

**Game Day Exercises:** Conduct game day exercises where the on-call team responds to a simulated outage without prior notice. This tests both the technical design and the operational response.

## Queue Replay Strategy After CBS Recovery

Replaying queued transactions after CBS recovery requires careful orchestration to prevent duplicates and ensure ordering:

**Priority Queuing:** Implement separate queues or priority levels: P0 (business-critical, time-sensitive wires), P1 (personal transfers with deadlines), P2 (bill payments), P3 (recurring transactions). Process P0 first.

**Idempotency Key Check:** Before replaying each transaction, check the core banking switch for the idempotency key. If the key already exists, the transaction was already processed — skip it and mark as complete in the queue.

**Batch Processing:** Replay transactions in batches of 100-500 to avoid overwhelming the recovering CBS. Monitor CBS health metrics during replay — if CPU or connection count spikes, reduce batch size.

**Failure Handling:** If a queued transaction fails during replay (e.g., insufficient funds due to intervening transactions), move it to a dead letter queue for manual review. Do not automatically retry failed replays.

**Reconciliation Verification:** After all queued transactions are replayed, run a reconciliation job that compares the queue entries with CBS transaction records to ensure nothing was lost or duplicated.

## Graceful Degradation Design Principles

Designing for graceful degradation requires thinking about every feature's dependency on external systems:

**Tier 1 — Must Always Work:** Authentication, authorization, session management, static content serving. These have no dependency on CBS and must remain functional during any outage.

**Tier 2 — Degrade Gracefully:** Balance inquiry (cached), transaction submission (queued), statement view (cached), notification delivery (delayed). These depend on CBS but can function with cached data or deferred processing.

**Tier 3 — Can Fail Temporarily:** Real-time fraud detection, cross-border transfers, loan applications, new account opening. These require CBS connectivity and cannot function without it. They should fail with clear error messages.

**Fallback Response Design:** Every degraded feature must return a response that includes: (1) the best available data, (2) a clear indication that the data may be stale or the operation is deferred, (3) a reference number if the operation was queued, and (4) an estimated time for full restoration.

**Audit During Degraded Mode:** Financial regulators require that every transaction — even those queued during outage — has a complete audit trail. The queue entry must capture: who initiated the transaction, when, what the intent was, and what the eventual outcome was. This audit trail must be maintained regardless of the outage.
