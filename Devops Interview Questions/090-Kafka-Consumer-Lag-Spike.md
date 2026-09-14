# 90. Kafka Consumer Lag Spike to 10 Million Messages

## Scenario

Your production Kafka cluster (3 brokers, 12 partitions on the `orders` topic) suddenly shows consumer lag spiking from 1,000 to 10 million messages in 15 minutes. Orders are being delayed — order processing is minutes behind real-time. The consumer group `order-service` has 5 consumers and should have ~12 partitions assigned (2-3 partitions each). The application is a Java/Scala service using the Kafka client. Monitoring shows the lag graph climbing steadily. Producers continue to produce at normal rates. Brokers look healthy (no errors in broker logs). You have 10 million messages backed up.

## Interviewer Question

"Kafka consumer lag suddenly increased from 1,000 to 10 million messages. Orders are delayed. The consumer group has 5 consumers and 12 partitions. Walk me through your detection, diagnosis, consumer health check, partition distribution, broker health check, scaling, and recovery."

## What I Should Think About

- Definition and measurement of consumer lag (end-offset minus committed offset)
- Detection: lag metrics via Kafka consumer group API, Prometheus kafka_exporter, Burrow
- Diagnosis: is it consumer-side (slow processing, exceptions) or broker-side (disk, network)?
- Consumer health: is the consumer alive? Are partitions assigned? Rebalances happening repeatedly?
- Partition distribution: 5 consumers vs 12 partitions → uneven load
- Whether processing throughput per consumer dropped (CPU, GC, DB calls)
- Check committed offsets vs actual offset
- Scaling strategy: add consumers, but note partition count caps parallelism (12 partitions max)
- Recovery: increase partitions? Dedicate consumers? Bump consumer parallelism? Offload processing
- Batch processing vs single-message processing tradeoffs

## Ideal Answer

**Phase 1: Detection**

Lag is the delta between the last committed offset and the current end offset per partition. Standard tooling:
- Prometheus `kafka_consumergroup_lag` metric (JMX/kafka-lag-exporter/Burrow)
- `kafka-consumer-groups --describe` for manual checks

**Phase 2: Diagnose consumer health**

Check if lag is due to:
1. Consumer down (partition reassignment to other consumers)
2. Consumer slow (processing time increased)
3. Consumer stuck (fetch loop dead, exception loop)
4. Rebalances happening every few seconds (lag accumulates during pause)

**Phase 3: Check broker health**

- Broker CPU/IO/disk
- Replication bandwidth
- Produce throughput (is producer still writing normally?)

**Phase 4: Recover**

- Fix consumer issue
- If throughput is the limit: add consumers (up to 12 = partition count), increase consumer concurrency per partition (parallel processing), etc.

## Architecture

```
  PRODUCERS → KAFKA (12 partitions) → CONSUMER GROUP "order-service"
                                        ┌──────────────────────────────┐
                                        │  5 consumers in group        │
  ┌────────┐    ┌────────────────┐      │                              │
  │order-api│───▶│ broker 1       │      │  consumer-1 → partitions     │
  └────────┘    │ p0 p3 p6 p9    │──┐   │    0,1,2                      │
  ┌────────┐    ├────────────────┤  │   │  consumer-2 → partitions     │
  │payment │───▶│ broker 2       │  └──▶│   3,4,5                       │
  └────────┘    │ p1 p4 p7 p10   │      │  consumer-3 → partitions      │
  ┌────────┐    ├────────────────┤  └──▶│   6,7                         │
  │inventory│──▶│ broker 3       │      │  consumer-4 → partitions      │
  └────────┘    │ p2 p5 p8 p11   │      │   8,9                         │
                └────────────────┘      │  consumer-5 → partitions      │
                                        │   10,11                       │
                                        │  1 unused consumer (idle)     │
                                        │                               │
                                        │  LAG: 10,000,000             │
                                        └──────────────────────────────┘

  LAG GRAPH:
  10M ┤                                  ◀── currently climbing
  5M  ┤                        ╱
  1M  ┤                ╱───────
  1K  ┤  ════════════
      └──────────────────────────────────────
       14:00    14:15    14:30    14:45
                ↑ spike starts (why?)
```

## Investigation

**Step 1: Confirm lag and identify which partitions**
```bash
# List consumer groups
kafka-consumer-groups.sh --bootstrap-server kafka:9092 --list

# Describe group with lag
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --describe --group order-service

# Output:
# TOPIC  PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# orders 0          1234567          2345678         1111111
# orders 1          1234567          2345678         1111111
# ...
# Focus: is lag across ALL partitions (global slowdown)
# or a FEW partitions (one consumer stuck)?"]
```

**Step 2: Check consumer status**
```bash
# Are consumer members active?
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --describe --group order-service --state

# Check for repeated rebalances
# Look for pattern: OK... REBALANCING... OK... REBALANCING

# Check consumer group membership over time
# If partition count assigned to consumers keeps changing → rebalance loop
```

**Step 3: Check consumer processing indicators**
```bash
# Metrics from JMX/consumer application:
# - records-consumed-rate
# - records-lag-max
# - fetch-rate
# - bytes-consumed-rate
# - commit-latency
# - If records-consumed-rate dropped while produce-rate unchanged → consumer bottleneck

# Check app-side: DB latency, CPU, GC
jcmd <pid> GC.heap_info
jstack <pid> | grep -A5 "kafka"
top -H <pid>  # thread CPU usage
```

**Step 4: Check broker health**
```bash
# Broker metrics
# - CPU utilization per broker
# - Disk I/O (kafka has local disk per broker)
# - Network throughput (in/out)
# - Fetch requests latency

# Check broker logs for errors
grep -i "error\|warn" /var/log/kafka/server.log | tail -50

# Check under-replicated partitions
kafka-topics.sh --bootstrap-server kafka:9092 \
  --describe --under-replicated-partitions
```

**Step 5: Check consumer configuration**
```bash
# Check max.poll.records, fetch.max.bytes, session.timeout.ms
# Large max.poll.records + slow processing → session timeout → rebalances
```

## Commands

```bash
# Get all consumer lag per topic/partition
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --describe --group order-service --members --verbose

# Check the last committed offset
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --describe --group order-service --members

# View topic end offsets
kafka-run-class.sh kafka.tools.GetOffsetShell \
  --broker-list kafka:9092 --topic orders --time -1

# Check under-replicated partitions
kafka-topics.sh --bootstrap-server kafka:9092 \
  --describe --under-replicated-partitions

# Increase consumer parallelism by adding consumers
# (up to 12 for 12 partitions)
kubectl scale deployment/order-consumer --replicas=12

# Configure consumer for faster consumption (increase max.poll.records)
# consumer.properties:
#   max.poll.records=500
#   fetch.min.bytes=1024
#   max.poll.interval.ms=300000
#   session.timeout.ms=45000
#   enable.auto.commit=false (manual commits in batches)

# If partition count is the bottleneck and messages are heavy:
# Increase topic partitions NOW
kafka-topics.sh --bootstrap-server kafka:9092 \
  --alter --topic orders --partitions 24
```

## Root Cause

1. **Slow downstream dependency** — each order message invokes a payment API call that was timing out (5s→30s). Processing time per message increased 6x, so consumers barely kept up
2. **Consumer parallelism mismatch** — 5 consumers across 12 partitions; 2 consumers were idle-with-partitions assignment uneven resulting in unequal load
3. **Repeated rebalances** — consumers timing out triggers rebalance; each rebalance pauses ALL consumption (~30s each), amplifies lag
4. **Batch processing missing** — consuming message-by-message with I/O per message
5. **Lag alerting absent** — no alarm when lag exceeded 10,000

## Immediate Mitigation

```bash
# 1. Stop the bleeding: pause production of the least-critical messages
#    (rate-limit producer for low-priority topics, not orders)

# 2. Add consumers up to partition count
kubectl scale deployment/order-consumer --replicas=12

# 3. If payment API is the bottleneck:
#    - Enable circuit breaker to fail fast instead of hanging
#    - Increase payment API throughput (scale it up)

# 4. Reduce processing time per message temporarily
#    - Bypass payment verification for risk-under-threshold orders
#    - Increase max.poll.records + batch DB writes

# 5. Monitor lag dropping; once it stabilizes, keep steady

# 6. If messages are getting stale and order SLA is 30 min:
#    - Consider skipping non-critical enrichments (geolocation)
#    - Priorities: process most urgent orders first
```

## Permanent Fix

1. **Match consumer count to partitions** (12 for 12 partitions) and monitor assignment balance
2. **Implement backpressure-aware consumption**: batch processing with parallelism, DB batch inserts
3. **Use circuit breakers and timeouts** for downstream calls so a slow dependency doesn't freeze consumption
4. **Correct consumer config**: `max.poll.records` tuned to avoid session timeouts; enable async commit with `enable.auto.commit=false` + manual batched commits
5. **Add lag-based autoscaling** on the consumer group (KEDA ScaledObject on lag)
6. **Add lag alerts**: warning at 10,000, critical at 100,000
7. **If long-term throughput is higher than 5-consumer capacity, consider two consumer groups** splitting work by domain (orders-enrichment, orders-payment)
8. **Monitor consumer rebalance frequency** — alerts if >1 rebalance per 15 min

## Monitoring

```bash
# Metrics via Prometheus kafka_exporter / Burrow:
# - kafka_consumergroup_lag (per group/topic)
# - kafka_consumergroup_current_offset
# - kafka_consumergroup_uncommitted_offsets
# - kafka_consumer_fetch_manager_records_consumed_total (rate)
# - kafka_consumer_coordinator_rebalance_metrics
# - Burrow lag evaluation: OK / WARNING / ERROR / STALLED

# Alerts:
# - Lag per partition > 10,000 → WARNING page
# - Lag per partition > 100,000 → CRITICAL page
# - Lag increasing (trend up for 10 min) → WARNING
# - Rebalance events > 2 / 15 min → WARNING
# - Consumer not committing offsets for 5 min → ERROR

# Set up Burrow (LinkedIn) for more precise lag evaluation
# Kafka lag exporter: https://github.com/danielqsj/kafka_exporter
```

## Security

- Only authorized teams should be able to `--alter --partitions` or scale consumer groups
- Access to `kafka-consumer-groups` shell should be restricted (admins only)
- TLS + SASL for all consumer connections (scram/oauth) — ensure consumer config uses proper ACLs
- Monitor for unauthorized consumer groups joining topics (unknown groups = attack indicator)
- Audit logs of admin commands (`kafka-acls.sh` show ACLs)

## Production Considerations

- **Scalability ceiling**: consumer parallelism capped by partition count. Plan partition count up front (or use keyed = ordered topics)
- **Cost**: extra consumers cost compute; balance with lag SLA
- **HA**: at least 3 brokers, replication factor ≥ 3 for topics to survive broker loss
- **Reliability**: exactly-once vs at-least-once — define and match consumer config
- **Operational**: runbook for lag incident; document max.poll.records tuning process
- **Compliance**: order processing SLA must be tracked; stale orders may violate SLAs — have metrics tied to business impact

## Senior-Level Answer

"I'd first confirm the lag is real using `kafka-consumer-groups --describe`, identifying whether lag is uniform or concentrated on specific partitions. Then I'd check for rebalances — a consumer timing out causes repeated 30-second pauses that let lag explode. The root cause is usually downstream latency (payment API timeouts) making per-message processing too slow, compounded by 5 consumers vs 12 partitions and missing batch processing. Immediate fix: add consumers to 12, enable circuit breakers on the slow dependency so consumers fail fast, and batch the DB writes. Permanent: lag-based autoscaling with KEDA, Burrow-based alerts on lag thresholds, and careful tuning of `max.poll.records`/`max.poll.interval.ms` to avoid session-timeout rebalance loops."

## Architect-Level Answer

"This is a throughput-vs-lag capacity problem. The commit: we need consumer capacity ≥ producer rate + headroom. The partition count caps parallelism, so the real architecture question is: is the `orders` stream workload balanced enough, or should we shard into multiple topics (orders-payments, orders-enrichment) each independently consumable and scalable? I'd adopt KEDA ScaledObject on lag for elastic autoscaling. For robustness, add a dead-letter queue for poison messages that currently just retry forever. Monitoring must include Burrow with its OK/WARNING/ERROR/STALLED semantics because it catches not just magnitude but trend. We should also model the business SLA as a lag budget: converting lag to minutes behind (lag / consumption rate) so alerts are business-meaningful. Finally, batch processing and fan-out architecture produce the sustainable answer — the 5-consumer design was undersized for vertical throughput."

## Follow-Up Questions

1. "Why can't you just scale consumers beyond the partition count, and how do you decide the right partition count up front?"
2. "Explain how `enable.auto.commit` interacts with at-least-once semantics and the risk of committing offsets before processing completes."
3. "You detect that lag is growing fastest in a single partition. What does that tell you and what do you do?"
4. "Describe how KEDA scales consumers based on lag — what metric and what ScaledObject config?"
5. "How would you drain a 10-million-message backlog without overwhelming downstream services?"