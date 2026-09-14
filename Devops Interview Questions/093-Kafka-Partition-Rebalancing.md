# 93. Kafka Partition Rebalancing Issues

## Scenario

A financial transaction processing system uses Kafka with a consumer group of 8 consumers. Every time a consumer joins or leaves (scale up, scale down, crash, or instance flappy), the consumer group triggers a rebalance. During rebalance, ALL consumers stop processing for 2 minutes. This means financial transactions are NOT processed for 2 minutes each time. This is unacceptable for a system with strict latency SLAs (p99 < 500ms). Rebalances happen frequently because an autoscaler and spot instances cause churn. You need to minimize rebalance impact.

## Interviewer Question

"During consumer group rebalancing, all consumers stop processing for 2 minutes — every time a consumer joins or leaves. Financial transactions can't tolerate this pause. How do you minimize rebalancing impact?"

## What I Should Think About

- Why rebalances stop all consumers (eager rebalance / stop-the-world)
- Rebalance protocol differences: eager vs incremental cooperative (KIP-429)
- Static membership (group.instance.id — KIP-345) to survive transient restarts without rebalance
- Session timeout tuning, heartbeat tuning
- How partition assignment strategy affects rebalance scope
- Reducing consumer churn (stable instances, no aggressive autoscaling)
- Rebalance time: how long assignment takes; `rebalance.timeout.ms`/`max.poll.interval.ms`
- Handling graceful vs ungraceful shutdown
- Batch commit + commit-on-revoke
- Cooperative rebalancing (Confluent) minimizes stop-the-world
- Watch `__consumer_offsets` and coordinator behavior

## Ideal Answer

**Why it happens:** Default consumer uses **eager rebalancing**: when the group sees any consumer join/leave, the coordinator revokes ALL partitions from ALL consumers (all stop), re-allocates, then consumers resume. So every membership change = full stop-the-world.

**Mitigations:**

1. **Reduced churn (biggest win)**
   - Use stable instance lifetimes (pin instances, avoid aggressive HPA in/out)
   - Spot instance + autoscaler = fragile churn
   - If autoscaling is mandatory, scale coarsely (add big chunks, not 1 at a time) and scale down slowly (cooldowns)

2. **Static membership (KIP-345)**
   - Set `group.instance.id` per consumer → during transient restarts, Kafka treats it as the same member, so NO rebalance
   - Requires at least one consumer in the group to be present (graceful restart avoids rebalance)

3. **Cooperative incremental rebalancing (KIP-429)**
   - Consumers adopt the "cooperative sticky assignator"; on membership change only the affected partitions are revoked/reassigned; the rest keep consuming → near-zero pause

4. **Tuning timeouts**
   - Increase `session.timeout.ms` and `heartbeat.interval.ms` to tolerate transient network blips without fake rebalance
   - Increase `max.poll.interval.ms` to avoid rebalance from slow poll

5. **Graceful shutdown**
   - On SIGTERM, call `consumer.close()` — triggers a smooth rebalance with fewer repeated steps

6. **Assignment policy**
   - Use `cooperative-sticky` + `range`/`roundrobin` consistent policy across all consumers

## Architecture

```
  EAGER REBALANCE (DEFAULT - STOP THE WORLD):
  ┌────────┐┌────────┐┌────────┐┌────────┐  consumers
  │ C1▣    ││ C2▣    ││ C3▣    ││ C4▣    │  ▣=consume
  └────────┘└────────┘└────────┘└────────┘
        │ C3 leaves (crash)
        ▼
  ┌────────┐┌────────┐┌────────┐┌────────┐
  │ C1◻    ││ C2◻    ││ C4◻    ││ (none) │  ◻=paused - ALL STOP
  └────────┘└────────┘└────────┘└────────┘
        │  coordinator reassigns ALL partitions
        ▼
  ┌────────┐┌────────┐┌────────┐┌────────┐
  │ C1▣    ││ C2▣    ││ C4▣    ││        │  resumed
  └────────┘└────────┘└────────┘└────────┘
  2 minutes of NOTHING processed.

  COOPERATIVE REBALANCE (KIP-429 - INCREMENTAL):
  ┌────────┐┌────────┐┌────────┐┌────────┐
  │ C1▣    ││ C2▣    ││ C3▣    ││ C4▣    │
  └────────┘└────────┘└────────┘└────────┘
        │ C3 leaves
        ▼
  ┌────────┐┌────────┐┌────────┐┌────────┐
  │ C1▣    ││ C2▣    ││ C4▣    ││        │  C1,C2 keep consuming
  └────────┘└────────┘└────────┘└────────┘      (only C3's partitions
        │ reassign only vacated partitions         are handed off)
        ▼
  C1 gains 1 partition, C4 gains 1. No global stop.

  STATIC MEMBERSHIP (KIP-345):
  consumer joins with group.instance.id="consumer-1"
  ── during a rolling restart the group coordinator
     recognizes same instance → no rebalance needed
```

## Investigation

**Step 1: Confirm rebalance frequency and duration**
```bash
# Metrics from consumer client:
# - kafka_consumer_coordinator_rebalance_total
# - kafka_consumer_coordinator_assigned_partitions
# - kafka_consumer_coordinator_rebalance_latency_avg

# Alternatively watch logs for:
grep -i "rebalance started\|rebalancing\|revoked\|assigned" app.log | tail -100

# If you see a pattern of:
#   ... assigned partitions [0,1]
#   ... revoked partitions     ← soon followed by reassignment → rebalance loop
```

**Step 2: Check consumer count stability**
```bash
# Frequently joining/leaving? → flapping
kubectl get pods -l app=tx-consumer -w   # watch for CrashLoopBackOff/restarts
kubectl describe pod tx-consumer-xyz | grep -i "restart"
# Check HPA activity
kubectl get hpa tx-consumer -o yaml
```

**Step 3: Check rebalance protocol / config**
```bash
# The instanceof coordinator metrics show which protocol in use:
# kafka_consumer_coordinator_join_rate (eager join)
# If cooperative, you'd see a "rebalance.max.retries" and cooperative protocol
# in the client logs
```

**Step 4: Check session/heartbeat settings**
```bash
# If session.timeout.ms is too low (e.g. 3000) → transient network hiccup
# causes a false rebalance
grep -i "session\|heartbeat" consumer.properties
```

**Step 5: Examine coordinator and __consumer_offsets**
```bash
# Identify group coordinator
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --describe --group tx-group --state

# Look at consumer group state transitions
# Check if "STABLE" persists or flips to "PREPARING_REBALANCE" often
```

## Commands

```bash
# Enable cooperative-sticky rebalance (same on ALL consumers):
# consumer.properties:
#   partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor
#   rebalance.timeout.ms=30000
#   session.timeout.ms=45000
#   heartbeat.interval.ms=15000
#   max.poll.interval.ms=300000

# Static membership (each consumer gets a unique instance.id):
# consumer.properties:
#   group.instance.id=tx-consumer-${HOSTNAME}   # stable value!
#   session.timeout.ms=45000

# Graceful shutdown in code (Java):
# consumer.wakeup();
# consumer.close(Duration.ofSeconds(30));

# Check rebalance events and group state
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --describe --group tx-group --state

# Check consumer group's coordinator and assignments
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --describe --group tx-group --members --verbose

# If autoscaling is blurring the group count, consider coarser scaling:
kubectl patch hpa tx-consumer -p '{"spec":{"behavior":{
  "scaleDown":{"stabilizationWindowSeconds":300}}}}'
```

## Root Cause

1. Consumer churn (autoscaler + spot instances) → frequent join/leave
2. Default **eager rebalance protocol** → global stop-the-world during each
3. Overly tight `session.timeout.ms` → network blips triggered phantom rebalances
4. No static membership → even rolling restarts triggered rebalances
5. No graceful close on SIGTERM → abrupt revocations

## Immediate Mitigation

```bash
# 1. Reduce churn NOW: disable aggressive autoscaling
kubectl scale deployment/tx-consumer --replicas=8
kubectl autoscale deployment/tx-consumer --min=8 --max=12 --cpu-percent=70

# 2. Make the group more tolerant of transient blips
#   consumer.properties:
#     session.timeout.ms=45000
#     heartbeat.interval.ms=15000

# 3. Use static membership so restarts aren't rebalances
#   group.instance.id=tx-consumer-pod-<uid>   (stable across restarts)

# 4. If a consumer is stuck in a rebalance loop, restart the consumer group coordinator
#    Or restart offending consumer pod(s) to resync state

# 5. In the worst case, temporarily move to synchronous processing handoff:
#    - Have each consumer lock its partitions entirely and never relinquish
#    - MANUAL partition assign (assign()) instead of subscribe() temporarily
```

## Permanent Fix

1. Adopt **cooperative-sticky assignor** (`CooperativeStickyAssignor`) — incremental rebalance, no global pause
2. **Static membership** (`group.instance.id`) across the fleet
3. Stable instances: prefer stable EC2/on-prem, avoid spot for critical consumers
4. Coarse autoscaling + cooldown windows to avoid oscillation
5. Graceful shutdown (catch SIGTERM → `consumer.close()`)
6. Tuned timeouts: `session.timeout.ms=45000`, `heartbeat.interval.ms=15000`, `max.poll.interval.ms` appropriate for workload
7. Prefer **Kafka Streams/DSL** where stateful consumers and rebalance-aware state (Changelog topics) are needed — rebalance is handled gracefully
8. Monitor group state: STABLE vs flapping

## Monitoring

```bash
# Consumer metrics on Prometheus (kafka_exporter / JMX):
# - kafka_consumer_group_members
# - kafka_consumer_coordinator_rebalance_total
# - kafka_consumer_coordinator_rebalance_latency_avg
# - kafka_consumer_coordinator_rebalance_metrics
# - kafka_consumer_fetch_manager_records_lag_max

# Alerts:
# - rebalance > 2 per 15 minutes → WARNING
# - rebalance duration > 30s → WARNING
# - group members change without deploy → WARNING
# - lag spike while rebalancing → ERROR if over business SLA

# Dashboard:
# - group size over time vs rebalance events
# - rebalance timing overlayed with lag
```

## Security

- `group.instance.id` shouldn't leak instance names/hostnames if that's sensitive
- ACLs on consumer group: prevent unauthorized members joining (someone could silently join/leave and cause rebalances)
- Restrict `alter` on consumer groups
- Authentication (SASL) mandatory for consumer registration to prevent eavesdropping on assignment

## Production Considerations

- **HA**: cooperative rebalancing + static membership are resilience features; deploy in that order
- **Scalability**: autoscaling based on lag (KEDA) works well with cooperative rebalance
- **Reliability**: financial stream uses stateful processing? A state store (rocksDB) + changelog topic is the correct companion to rebalances
- **Cost**: static membership + stable nodes may increase cost vs aggressive spot autoscaling — tradeoff justified by SLA
- **Compliance**: transaction processing SLA requires documented rebalance mitigation
- **Operational**: runbook for consumer churn; ensure consumer version parity across pods (mismatched clients cause rebalances)

## Senior-Level Answer

"Rebalancing causes stop-the-world because the default protocol is eager: any join/leave revokes every partition from every consumer. I'd reduce churn at the source by stabilizing instances and adding autoscaling cooldowns. Then I'd enable `CooperativeStickyAssignor` (KIP-429) so rebalances are incremental — only the vacated partitions move, the rest keep consuming. Static membership via `group.instance.id` (KIP-345) makes a rolling restart seamless. Tuning `session.timeout.ms` up and `max.poll.interval.ms` avoids phantom rebalances from GC pauses or slow polls. And graceful `close()` on SIGTERM prevents abrupt revocations. For stateful financial processing, Kafka Streams automatically handles state migration during rebalance so this class of issue is managed by the framework."

## Architect-Level Answer

"This is a rebalance-design problem that belongs in the platform layer. I'd standardize the consumer baseline: cooperative incremental rebalancing enabled everywhere, static membership, generous timeouts, graceful shutdown hooks, and group-membership monitoring. The platform should also define scaling behavior: KEDA-based lag autoscaling with cooldown windows instead of naive HPA that flaps. For stateful transactions, adopt Kafka Streams so the framework owns state-store migration, preventing data loss during rebalance. I'd also set SLOs on group health — stable group STABLE state and bounded rebalance frequency — enforced by alerts. The deeper architectural answer: isolate cardinality — order-critical topics on dedicated stable consumer groups, while best-effort topics use cheap autoscaled groups, so expensive rebalancing applies only where latency demands it."

## Follow-Up Questions

1. "What's the exact difference between the eager protocol and the cooperative incremental protocol in terms of partition revocation?"
2. "How does static membership (group.instance.id) avoid rebalance during a crash, and what's its limitation?"
3. "If you have a stateful consumer (materialized aggregation), how does rebalancing handle the state store migration?"
4. "How do you choose session.timeout.ms, heartbeat.interval.ms, and max.poll.interval.ms for a workload with variable processing time?"
5. "Explain how the rebalance timeout works in KIP-429 and what a consumer does during the revocation phase."