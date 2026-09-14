# 91. Kafka Broker Failure - Partition Availability

## Scenario

One of 3 Kafka brokers crashed unexpectedly (B2). The topic `payments` has replication factor 2, 9 partitions. Before the crash, the leader distribution was: broker1→4 partitions, broker2→3 partitions, broker3→2 partitions. After B2 went down, 5 partitions lost their leader because some partitions had both replicas on B2+B1 (or B2+B3 depending on placement). Producers start receiving `NotEnoughReplicasException` / `NOT_ENOUGH_REPLICAS`. Some consumers are receiving messages with no leader. The cluster is on version 3.6, using KRaft mode with 3 controller nodes. Monitoring shows broker2 is unresponsive to health checks. The team is panicking but messages ARE flowing (partially).

## Interviewer Question

"One of 3 Kafka brokers crashed. Replication factor is 2. Some partitions lost their leader. Producers are getting NotEnoughReplicas errors. How does Kafka handle broker failure? How do you recover and ensure data integrity?"

## What I Should Think About

- How Kafka handles broker failure: controller detects, partitions reassign leadership to in-sync replicas
- Under-replicated partitions concept
- `min.insync.replicas` vs replication factor — the source of NotEnoughReplicas errors
- Leader election: unclean vs clean; ISR maintenance
- If a partition had 2 replicas and one was on the dead broker, and the surviving ISR has only 1 replica, `min.insync.replicas=2` blocks writes
- Recovery: bring broker2 back; verify data; clean restored brokers
- Data integrity: ISR guarantees means data on ISR replicas is complete
- Risk of unclean leader election (data loss)

## Ideal Answer

**How Kafka handles broker failure in normal operation:**

1. **Detection**: ZooKeeper/KRaft controller detects broker heartbeat timeout
2. **Leader re-election**: For each partition whose leader was on the failed broker, the controller selects a new leader from the in-sync replica (ISR) set
3. **Under-replication**: Partitions with replicas on the dead broker become `UnderReplicated` — the ISR shrinks to remaining replicas
4. **Writes continue** only if `min.insync.replicas` can be satisfied
5. **Producers with `acks=all`**: fail if ISR < `min.insync.replicas` — this is the `NotEnoughReplicas` error

**Why NotEnoughReplicas even with RF=2:** RF=2 means 2 copies. If RF=2 and `min.insync.replicas=2`, after a broker loss you have 1 ISR replica → writes blocked. That's the tradeoff: stronger durability guarantees vs availability.

**Recovery:**
- Restart broker2 (or replace the node)
- Broker rejoins cluster; data catches up from leader (or follows leader)
- When it re-contributes ISR, partitions return to healthy state
- Under-replicated partition count returns to 0

## Architecture

```
  KAFKA CLUSTER (3 brokers, KRaft, 3 controllers)

  BEFORE (normal):
  ┌────────────┐  ┌────────────┐  ┌────────────┐
  │ Broker 1   │  │ Broker 2   │  │ Broker 3   │
  │ Leaders:   │  │ Leaders:   │  │ Leaders:   │
  │  p0,p2,p5  │  │  p1,p4,p7  │  │  p3,p6,p8  │
  │ Replicas:  │  │ Replicas:  │  │ Replicas:  │
  │  + p1,p3   │  │  + p0,p8   │  │  + p2,p4   │
  │  etc       │  │  etc       │  │  etc       │
  └────────────┘  └────────────┘  └────────────┘

  AFTER B2 CRASH:
  ┌────────────┐  ┌──────────────┐  ┌────────────┐
  │ Broker 1   │  │ Broker 2     │  │ Broker 3   │
  │ Leaders:   │  │  DEAD        │  │ Leaders:   │
  │  p0,p1,p2, │  │              │  │  p3,p4,p5, │
  │  p7,p8     │  │              │  │  p6        │
  │  (re-      │  │              │  │ (re-       │
  │  elected)  │  │              │  │  elected)  │
  └────────────┘  └──────────────┘  └────────────┘

  Partitions p1, p4, p7 lost leaders on B2.
  - If p1's ISR had only B2 (because B1+B2 but B1... )
  - If ISR after failure has 1 member hitting RF-1:
      producers with acks=all + min.insync.replicas=2:
      → NOT_ENOUGH_REPLICAS (writes blocked for that partition)

  UNDER-REPLICATED PARTITIONS (URP):
  ┌──────────────────────────────────────────┐
  │ topic=payments  partition=1              │
  │   leaders: broker1  replicas:[1,2]       │
  │   isr: [1]  (broker2 gone)  URP=Yes      │
  │   min.insync.replicas=2 → writes blocked │
  └──────────────────────────────────────────┘
```

## Investigation

**Step 1: Identify cluster and leadership state**
```bash
kafka-topics.sh --bootstrap-server kafka:9092 --describe \
  --topic payments

# Output shows:
# Topic: payments  Partition: 1  Leader: -1  Replicas: [1,2]  Isr: [1]
# Leader: -1 means NO leader (available) for that partition

# Check all under-replicated partitions
kafka-topics.sh --bootstrap-server kafka:9092 \
  --describe --under-replicated-partitions
```

**Step 2: Check brokers and ISR**
```bash
# List brokers
kafka-brokers.sh --bootstrap-server kafka:9092 --list

# Check partition leadership (leader mapping)
kafka-topics.sh --bootstrap-server kafka:9092 --describe \
  --topic payments | grep Leader
```

**Step 3: Confirm producer errors**
```bash
# Look at producer logs:
grep -i "NOT_ENOUGH_REPLICAS\|UnknownLeaderEra\|LeaderNotAvailable" \
  app.log | tail -50
```

**Step 4: Check controller status and broker logs**
```bash
# KRaft: check controller
kafka-metadata-shell.sh --snapshot /var/lib/kafka/metadata/__cluster_metadata-0.log
# Or check controller logs on each node
grep -i "elected\|controller" /var/log/kafka/controller.log | tail -100

# Broker failed node's OS logs:
dmesg | grep -i "kafka\|oom" | tail
journalctl -u kafka --since "1 hour ago" | grep -iE "error|fatal"
```

**Step 5: Check data on surviving replicas**
```bash
# Verify offsets on each surviving broker for the affected partitions
kafka-run-class.sh kafka.tools.GetOffsetShell \
  --broker-list kafka:9092 --topic payments --partitions 0,1,2
```

## Commands

```bash
# Check topic health
kafka-topics.sh --bootstrap-server kafka:9092 --describe --topic payments
kafka-topics.sh --bootstrap-server kafka:9092 \
  --describe --under-replicated-partitions

# Check potential for unclean leader election (data loss risk)
kafka-configs.sh --bootstrap-server kafka:9092 \
  --entity-type topics --entity-name payments --describe | grep unclean

# Monitor in-sync replicas
kafka-topics.sh --bootstrap-server kafka:9092 --describe \
  --topic payments --unavailable-partitions

# If leader is -1 (no leader), recover:
# 1. Restart the dead broker or bring replacement
systemctl start kafka
# 2. Verify it joins and ISRs rebuild
kafka-topics.sh --bootstrap-server kafka:9092 \
  --describe --topic payments

# Check consumer progress (offsets remain in Kafka log, consumers continue)
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --describe --group payment-processor

# Force leader election for leaderless partitions (if controller didn't auto-elect)
# (only if auto.leader.rebalance.enable / unclean failover allowed)
kafka-leader-election.sh --bootstrap-server kafka:9092 \
  --topic payments --partition 1 --election-type preferred
```

## Root Cause

1. **RF=2 with `min.insync.replicas=2`** — broker loss reduces ISR below min → writes blocked by design (data integrity over availability)
2. **Improper replica placement** — replicas of the same partition ended up only on B2+B1, meaning some partitions shared a broken pair
3. **Possible OOM/kernel issue on B2** — need to check node resources before restart
4. **Missing auto leader re-election config** (if leaders not automatically re-elected) — in some setups preferred leader needs manual trigger
5. **Data on surviving ISR is authoritative** — no data was lost IF `unclean.leader.election.enable=false`

## Immediate Mitigation

```bash
# 1. Try to restart the broker (if hardware OK):
systemctl restart kafka   # or kafka-server-start.sh

# 2. If broker2 is gone permanently:
#    - Decommission it
kafka-broker-api-versions.sh --bootstrap-server kafka:9092
#    - Remove from cluster via kafka-configs / kafka-controller (KRaft)
#    - Add replacement broker node with same broker.id
#    - Let replicas rebuild (reassign if needed)

# 3. If a partition has NO leader (Leader: -1) and unclean election is DISABLED:
#    - Only recovery is restoring the missing replica.
#    - Optionally enable unclean election CONSIDERING data loss:
kafka-configs.sh --alter --entity-type topics --entity-name payments \
  --add-config unclean.leader.election.enable=true

# 4. Ensure min.insync.replicas is tolerant:
#    For RF=2 → min.insync.replicas=1 only if you accept data-loss risk.
#    Usually keep 2 and accept limited writes; or move to RF=3.
```

## Permanent Fix

1. **RF=3 for critical topics** — survives 1 broker failure without under-replication
2. **Keep `min.insync.replicas` ≤ RF-1** (e.g., RF=3, min.insync=2) — writes continue with 2 ISR replicas and durability still high
3. **Proper replica placement** — racks/availability zones; avoid co-locating replicas on the same physical host
4. **Monitoring CAUGHT the outage** but the alerting must respond to `Leader: -1` and URP count
5. **Node capacity/health checks** — monitor disk, memory, and JVM before failures
6. **Documented broker rotation/decommission runbook**
7. **Graceful shutdown configuration** for brokers (controlled rollover)
8. **Enable `auto.leader.rebalance.enable` and preferred replica leader election on a schedule**

## Monitoring

```bash
# Key metrics:
# - kafka_server_replica_manager_underreplicatedpartitions (Kafka metric)
# - kafka_server_kafkaserver_brokertopicmetrics_underreplicatedpartitionscount
# - OnlinePartitionCount
# - OfflinePartitionCount (Leader: -1)
# - kafka_controller_kafkacontroller_leadercount
# - ActiveControllerCount
# - Broker CPU, disk IO, network, JVM GC

# Alerts:
# - UnderReplicatedPartitions > 0 for 10 min → WARNING
# - OfflinePartitions > 0 → CRITICAL page
# - ActiveControllerCount != 1 → WARNING
# - Broker down (heartbeat lost) → CRITICAL
# - ISR shrink events → WARNING
```

## Security

- Broker credentials/TLS should be present; broker-to-broker communication encrypted
- ACLs on topics: producers/consumers get minimal permissions
- Decommissioned brokers must have keys/ACLs revoked immediately
- Control-plane access (KRaft controller) restricted; admin API audited
- Monitor for unauthorized producers joining after broker recovery (data injection risk)

## Production Considerations

- **HA**: RF=3 is the baseline for production-critical topics; RF=2 leaves you fragile exactly like this
- **RPO/RTO**: with default settings, RF=2 + min.insync=2 favors zero data loss over availability; document that tradeoff for business
- **Cost**: RF=3 triples storage + more replication bandwidth — justify
- **Compliance**: financial data requires durability — RF=3 with min.insync=2 is standard
- **Operational**: replicas rebuilding after broker recovery saturate network; stagger recovery
- **Scalability**: adding brokers rebalances partitions automatically but monitoring must catch skew
- **DR**: broker loss ≠ region loss; plan region-level DR separately (rack awareness, mirror via MirrorMaker or cluster linking)

## Senior-Level Answer

"Kafka self-heals broker failure via the controller: it detects the dead broker via heartbeat timeout, elects new leaders from the ISR, and marks partitions under-replicated. The real problem here is that RF=2 with `min.insync.replicas=2` means losing one broker drops ISR below the minimum → producers get exactly the `NOT_ENOUGH_REPLICAS` we see. That's durability-vs-availability by design. For recovery: restart or replace Broker 2, let ISR regenerate, confirm URP drops to 0. Data on the surviving ISR replicas is authoritative (assuming `unclean.leader.election.enable=false`). The permanent fix is RF=3 with min.insync=2 and proper rack awareness so a single node loss never blocks writes."

## Architect-Level Answer

"This incident shows the cost of RF=2. I'd mandate RF=3 for all mission-critical topics with `min.insync.replicas=2`, which tolerates one broker loss with no write interruption. Beyond redundancy, the design must include rack/zone-aware placement so replicas don't share failure domains, plus controller redundancy (3 KRaft controllers) so controller loss doesn't cascade. The operational piece is just as important: a decommissioning runbook, pre-flight disk/memory health checks, and automatic preferred-leader rebalance. For true disaster coverage, region-level mirroring (MirrorMaker 2 or Cluster Linking) protects against losing the whole cluster — broker failure is only the first layer. Finally, define and enforce per-topic durability requirements as code (Kafka ACLs + topic templates) so nobody creates an RF=2 topic by accident."

## Follow-Up Questions

1. "What is the exact semantic difference between `replication.factor` and `min.insync.replicas`, and how do they interact?"
2. "If `unclean.leader.election.enable=true` is needed for availability, how do you quantify the potential data loss and is it acceptable for payment data?"
3. "Explain how Kafka prefers to keep replica leaders balanced after broker recovery and when a leader rebalance occurs."
4. "How would you handle broker replacement with the same broker.id while old data replicas still exist?"
5. "Describe the difference between KRaft and ZooKeeper-based leader election in this failure scenario."