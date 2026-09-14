# 94. Kafka Dead Letter Queue Handling Strategy

## Scenario

A Kafka consumer processes customer account events. 5% of incoming messages fail to process due to malformed data (schema mismatches, missing fields, bad JSON). These failing messages continuously retry, blocking the partition. The consumer logs show the same errors repeatedly. Legitimate events behind a bad message are being delayed. The team wants a proper retry + DLQ strategy: retry transient failures with exponential backoff, route permanent failures to a DLQ topic, and alert when DLQ grows.

## Interviewer Question

"A Kafka consumer is failing to process 5% of messages due to malformed data. These keep retrying and blocking the partition. How do you implement a Dead Letter Queue (DLQ) strategy with retry, backoff, and alerting?"

## What I Should Think About

- Why retrying inside the consumer blocks the whole partition
- Distinguish transient vs permanent failures (counting, error classification)
- Retry strategies: in-memory retry vs retry queues vs separate retry topic
- Backoff: exponential with jitter; cap retries
- DLQ topic design (partition count, retention, schema)
- Why a per-message DLQ is better than blocking the partition
- Offset handling on permanent failure: commit offset but push message to DLQ — don't block
- Correlation ID / original partition/offset for replayability
- DLQ monitoring and alerting (lag on DLQ topic, DLQ consumer for inspection)
- Schema validation as preventive measure
- Handling of poison pills and deserialization errors

## Ideal Answer

**The core problem:** Retrying a bad message inside the consumer blocks the partition — Offset stays uncommitted, all subsequent messages wait. 
Better architecture:

1. **Classify failures**: transient (timeout, DB up) vs permanent (malformed JSON, schema violation)
2. **Retry transient failures with exponential backoff** — cap retries (e.g., 3 attempts)
3. **DlQ for permanent failures** — commit the message offset to keep the partition flowing, write the message (+ failure reason, headers) into a DLQ topic
4. **DLQ consumers** inspect/alert; remediation can replay back to main topic

**Retry implementation options:**
- In-consumer retry loop with backoff (careful: still blocks partition during backoff)
- **Retry topic** (`topic.retry`), messages reprocessed on a separate consumer walking the retry topic; produces less blockage
- Separate `retry` partition/consumer per processing step, exponential `ScheduledExecutorService`
- Standard pattern: main topic → consumer with limited in-memory retry → or forward to retry topic with backoff → after N failures → DLQ topic

**Backoff design:**
- Retry 1: wait 1s; Retry 2: wait 2s; Retry 3: wait 4s... apply full jitter
- Cap at, say, 5 attempts. After that → DLQ.

## Architecture

```
  MAIN FLOW:
  ┌──────────┐   ┌───────────────────────────────┐   ┌────────────┐
  │kafka topic│──▶│ CONSUMER (account-events)     │──▶│ DB / API   │
  │ account.evt│  │  process(record)              │   └────────────┘
  └──────────┘   │   ├─ transient error?          │
                 │   │    └─ retry (exponential+jitter)
                 │   │                 │ max 5 tries
                 │   │                 ▼
                 │   ├─ permanent failure? ──────┐
                 │   └─ success → commit         │
                 └───────────────────────────────┤
                                   FAILED ────────┤
                                                 ▼
                                   ┌─────────────────────────┐
                                   │ DLQ topic: account.evt.  │
                                   │        dead (12 partitions)│
                                   │  header: original_topic,  │
                                   │    original_partition,    │
                                   │    original_offset,       │
                                   │    error_class,           │
                                   │    error_message, retries │
                                   └───────────┬───────────────┘
                                               │
                              ┌────────────────▼────────────────┐
                              │ DLQ consumer / alerting          │
                              │  - log/serialize to S3 for dev   │
                              │  - alert if lag > threshold       │
                              │  - manual replay to main topic   │
                              └─────────────────────────────────┘

  RETRY TOPIC PATTERN (more resilient backoff):
  account.evt ──▶ consumer ──▶ success? ──▶ commit
                                  │ fail (transient)
                                  ▼
                        account.evt.retry (produce;
                        backoff via scheduled consumer poll)
                                  │ N retries exhausted
                                  ▼
                        account.evt.dead   (DLQ topic)
```

## Investigation

**Step 1: Confirm the pattern of failures**
```bash
kubectl logs -l app=account-consumer --tail=5000 \
  | grep -i "error\|failed" | awk '{print $NF}' | sort | uniq -c | sort -rn | head

# Categorize:
# - Deserialization errors (malformed JSON) → permanent
# - Timeout/connection errors (to DB) → transient
```

**Step 2: Check backlog and lag on the blocked topic**
```bash
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --describe --group account-consumer

# Look for high lag on SPECIFIC partitions — hint: a poison message at head
```

**Step 3: Check the offending message**
```bash
# Consume the partition to inspect the bad record
kafka-console-consumer.sh --bootstrap-server kafka:9092 \
  --topic account.evt --from-beginning --max-messages 5 \
  --property print.partition=true --property print.offset=true
# Examine for schema/field issues
```

**Step 4: Verify whether errors are retryable vs permanent**
```bash
# In consumer logs: same message key, same error, over and over → retry loop
grep "account-id=12345" app.log | tail -50   # see repeated identical failure
```

## Commands

```bash
# Create DLQ topic (reuse main topic partition count for parallelism)
kafka-topics.sh --bootstrap-server kafka:9092 \
  --create --topic account.evt.dead \
  --partitions 12 --replication-factor 3 \
  --config cleanup.policy=delete --config retention.ms=259200000

# Create retry topic
kafka-topics.sh --bootstrap-server kafka:9092 \
  --create --topic account.evt.retry \
  --partitions 12 --replication-factor 3

# Inspect DLQ contents
kafka-console-consumer.sh --bootstrap-server kafka:9092 \
  --topic account.evt.dead --from-beginning \
  --property print.headers=true \
  --max-messages 10

# Get DLQ lag (for alerting)
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --describe --group dlq-inspector

# Optional: cutover old malformed messages (offset reset for poison message)
# (do NOT blindly reset offsets; prefer DLQ-based approach)
```

## Root Cause

1. **No failure classification / no cap** — every failure retried forever in the consumer
2. **Blocking retry** — offset held uncommitted → partition stalls
3. **No DLQ design** — no place to park permanently-failing messages
4. **No schema registry/validation upstream** — producers could emit invalid records undetected
5. **No alerting/lag threshold** — the backlog grew without notice

## Immediate Mitigation

```bash
# 1. Stop the retry loop: fail fast and FAST-skip permanently invalid records
#    - Deploy consumer with retry cap + classification

# 2. Or, quick workaround: identify the poison message offset and reset the group
#    AFTER SAVING the payload to DLQ/S3
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --reset-offsets --group account-consumer --to-offset <offset> --topic account.evt

# 3. Manually drain by consuming ahead of the poison with isolation:
#    Use assign() with the partitions and skip the bad key

# 4. Ensure the consumer commits offsets for valid messages even when
#    some fail — commit valid processed offsets (manual). NEVER block all.
```

## Permanent Fix

1. **Failure classification + cap** (transient/Permanent; maximum attempts ~5)
2. **Retry topic with exponential backoff + jitter** (or Spring RetryTemplate w/ `scheduledExector`)
3. **DLQ topic with enriched headers** (original topic/partition/offset, error class, stack, attempt count) and appropriate retention
4. **Schema validation / Schema Registry** (Avro/JSON schema) at produce AND consume to prevent malformed data
5. **DLQ inspection tooling**: a `dlq-inspector` consumer that logs, stores to S3/object storage, and supports manual replay
6. **Replay pipeline**: ability to fix data in the DLQ and re-produce to the main topic with the same key (idempotency maintained)
7. **Monitor DLQ lag & error rate**; contribute to SLI/SLO

## Monitoring

```bash
# Metrics:
# - consumer messages processed / failed / retried
# - retry topic lag
# - DLQ topic lag (offset gap)
# - error rate per error class
# - retry attempts distribution

# Alerts:
# - DLQ lag > 100 → WARNING
# - DLQ lag > 1,000 or growing 10 min → CRITICAL
# - Error rate > 1% sustained → WARNING
# - Memory pressure in retry executor → WARNING

# Dashboard panels:
# - main topic per-partition lag
# - retry attempts (1..5)
# - DLQ lag by error class
# - replay activity
```

## Security

- DLQ messages may contain PII (account info) — apply retention/archival controls, encryption (at rest / in transit), access restrictions to the DLQ topic
- Header data should not leak stack traces with credentials
- DLQ admin access should be role-separated; only remediation people can replay
- Monitor for someone reading DLQ beyond authorization (ACLs)

## Production Considerations

- **Reliability**: DLQ is the safety valve that keeps the partition flowing
- **Scalability**: DLQ/retry topics share partition counts with the main topic to preserve ordering and parallelism; use separate hosts for DLQ consumer
- **Cost**: DLQ retention costs storage — balance retention (7 days) with replay needs
- **Compliance**: audit trail of replay actions — who, when, what (log all replay operations)
- **Operational**: replay runbook; DLQ consumer that pauses + removes the poison from the stream when needed

## Senior-Level Answer

"The whole point of a DLQ is to keep the healthy partition flowing. I'd classify failures as transient or permanent. Transient → retry with exponential backoff + jitter, capped at 5 attempts using a retry topic (so the main consumer doesn't block while backing off). Permanent → commit the offset, publish to a DLQ topic with headers containing the original topic/partition/offset and error details, then move on. A DLQ inspector consumer monitors lag, archives payloads to object storage, and supports safe replay after the data is fixed. Add schema registry validation at produce time to cut malformed input at the source. Track DLQ lag and error rate as SLO metrics with pageable alerts."

## Architect-Level Answer

"A DLQ strategy is part of a resilient event-driven architecture. I'd establish it as a platform-level standard: every critical consumer gets retry + DLQ topics with a common naming convention, enriched headers, and a shared replay mechanism. Schema validation sits at the edge — Schema Registry with compatibility checks prevents malformed events from ever entering the stream. The DLQ itself should feed an inspection pipeline (archive to S3, dashboard of error taxonomy) and a replay workflow. For ordering-sensitive systems we must preserve the invariant that a replayed message carries its idempotency key so downstream guards hold. Finally, DLQ growth becomes a product SLO metric, with pageable alerts, so '5% failures' shows up as a trend alert rather than a manual fire."

## Follow-Up Questions

1. "How would you design the retry topic so that backoff doesn't block the main consumer and preserves per-key ordering?"
2. "How do you distinguish transient from permanent errors programmatically, and how do you update classification when a previously-transient error becomes permanent?"
3. "What goes in the DLQ topic headers and how do you guarantee a safe, idempotent replay back to the main topic?"
4. "Explain how Schema Registry compatibility modes (FORWARD, BACKWARD, FULL) prevent malformed messages in the first place."
5. "How do you monitor the DLQ trend (not just absolute lag) to detect a systemic schema regression early?"