# 92. Message Duplication in Kafka Consumers

## Scenario

A banking application processes payment transactions from Kafka. Customers are reporting double charges. Investigation shows that the `payments` consumer group is processing duplicate messages. The consumer group uses `enable.auto.commit=true` with default settings (`auto.commit.interval.ms=5000`). When a consumer processing a message crashes or rebalances BEFORE the auto-commit happens, the message is redelivered → duplicate processing → double charges. The application writes payments to PostgreSQL with no idempotency key and no unique constraint. You must solve this properly: exactly-once or at-least-once with idempotent processing.

## Interviewer Question

"Consumers are processing duplicate messages, causing double payments. The consumer group uses auto-commit. How do you implement exactly-once or at-least-once semantics with idempotent processing in Kafka?"

## What I Should Think About

- Kafka delivery semantics: at-most-once, at-least-once, exactly-once
- Root cause of duplicates: auto-commit + rebalance/crash → offset not committed → redelivery
- Options to fix:
  1. At-least-once + idempotent processing (DB unique constraint, idempotency key)
  2. Exactly-once (EOS) via transactions + read_committed
  3. Manual commits after successful processing
- Transactional outbox pattern for reliable writes to DB + Kafka
- DB unique constraint as the ultimate guard
- Consumer rebalance and session timeout interplay
- Batch processing and commit strategies

## Ideal Answer

The default consumer gives **at-least-once** semantics (auto-commit). The problem: auto-commit happens every 5 seconds, regardless of whether processing completed. When a crash/rebalance occurs between "message processed" and "offset committed", the message is redelivered. With payments, retry of a non-idempotent operation = double charge.

**The fix: make processing idempotent.**

1. **At-least-once + idempotency key** (pragmatic, standard for payments):
   - Each order/payment event carries a unique `event_id` or `payment_request_id`
   - The DB has a UNIQUE constraint on that idempotency key (per payment)
   - On reprocessing, INSERT fails with unique violation → consumer treats as already-processed → skips and commits

2. **Alternatively exactly-once (EOS)**:
   - Enable transactions on producer + consumer config `isolation.level=read_committed`
   - Consumer processes and produces results to a downstream topic inside a Kafka transaction
   - Only commit offset through `transactional.id` __transaction_state
   - This gives E2E exactly-once ONLY if downstream storage also uses the transactional pattern
   - For DB writes, EOS with the classic transactional outbox pattern is more robust

3. **Manual commit** (avoid the blind spot):
   - `enable.auto.commit=false` → commit after processing completes
   - Commit in the finally or after batch is persisted

## Architecture

```
  CURRENT (duplicate-prone):
  ┌──────────────┐   ┌────────────────────────────┐   ┌──────────────┐
  │ Kafka topic  │──▶│ Consumer (auto.commit)     │──▶│ PostgreSQL   │
  │ payments     │   │  process msg → write DB    │   │ payment table│
  └──────────────┘   │  (every 5s auto-commit)    │   └──────┬───────┘
                     └───────────┬────────────────┘          │
                                 │ crash/rebalance BEFORE    │
                                 │ 5s commit                 │ ▼
                                 │ → msg re-delivered →      │ DOUBLE
                                 │ DB writes AGAIN           │ CHARGE
                                 └───────────────────────────┘

  FIXED: at-least-once + idempotency key
  ┌──────────────┐   ┌────────────────────────────┐   ┌──────────────┐
  │ Kafka topic  │──▶│ Consumer (manual commit)   │──▶│ PostgreSQL   │
  │ payments     │   │  event has idempotency_key │   │ payment table│
  └──────────────┘   │                            │   │ UNIQUE (key) │
                     │  INSERT ... ON CONFLICT    │   └──────┬───────┘
                     │  DO NOTHING                │          │
                     │  if conflict → skip, ack   │          │
                     │  commit offset AFTER done  │          │
                     └────────────────────────────┘          ▼
                     → redelivery is harmless:               no
                       idempotency key blocks duplicate       double charge

  EXACTLY-ONCE (EOS) with transactions:
  Producer.payments ──▶ Kafka (read_committed) ──▶ Consumer
                                                     ┌─ KTransaction
                                                     │ begin Tx (Kafka)
                                                     │ process
                                                     │ produce result → outbox
                                                     │ end Tx (commit offset)
```

## Investigation

**Step 1: Confirm duplicates are from redelivery**
```bash
# Check consumer group for rebalance events
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --describe --group payment-processor --state

# Look for recent rebalances in application logs
grep -i "rebalance\|rebalance started\|revoked" app.log | tail -50

# Check if any duplicate event_id is present
grep "payment_request_id" app.log | sort | uniq -c | sort -rn | head -20
```

**Step 2: Check consumer commit configuration**
```bash
# Check consumer config used:
# bootstrap.servers, group.id, enable.auto.commit, auto.commit.interval.ms
# - enable.auto.commit=true is the smoking gun
```

**Step 3: Check batch processing and commit timing**
```bash
# Measure average time from fetch to commit
# In app logs: log both "processing done" and "commit" events
# Look for gaps where commit lags behind world (5s auto-commit interval)
```

**Step 4: Check for exceptions mid-processing**
```bash
# Partially processed messages: DB write succeeded but
# downstream steps failed → offset not committed → redelivery
kubectl logs -l app=payment-consumer --tail=5000 | grep -i "error" | tail -30
```

## Commands

```bash
# Verify and find duplicates via the idempotency key
psql -c "SELECT payment_request_id, COUNT(*)
         FROM payments GROUP BY payment_request_id
         HAVING COUNT(*) > 1;"

# Check the current consumer commit interval setting
# consumer.properties:
#   enable.auto.commit=false
#   auto.commit.interval.ms=  (only relevant if auto-commit is on)
#   session.timeout.ms=10000
#   heartbeat.interval.ms=3000
#   max.poll.interval.ms=300000

# EOS consumer config for read_committed:
#   isolation.level=read_committed

# EOS producer config:
#   transactional.id=payment-processor-1
#   enable.idempotence=true
#   acks=all
```

## Root Cause

1. `enable.auto.commit=true` → offsets committed every 5s regardless of processing success
2. Crash/rebalance between processing and commit → redelivery
3. **Processing is NOT idempotent** — INSERT INTO payments has no unique key → second insert succeeds → double charge
4. (Secondary) batch processing across partitions without transactional grouping amodds to duplicate risk

## Immediate Mitigation

```bash
# 1. STOP bleeding: pause the consumer to prevent further double-payments
kubectl scale deployment/payment-consumer --replicas=0

# 2. Add a UNIQUE constraint on the payments table right now
psql -c "CREATE UNIQUE INDEX IF NOT EXISTS uq_payments_req_key
         ON payments(payment_request_id);"

# 3. Clean up existing duplicates
psql << 'EOF'
WITH dupes AS (
  SELECT payment_request_id,
         MIN(id) AS keep_id,
         ARRAY_AGG(id) AS all_ids
  FROM payments GROUP BY payment_request_id HAVING COUNT(*) > 1
)
DELETE FROM payments a USING dupes d
WHERE a.payment_request_id = d.payment_request_id
  AND a.id <> d.keep_id;
EOF

# 4. Restore the consumer (now idempotent via unique constraint)
kubectl scale deployment/payment-consumer --replicas=5
```

## Permanent Fix

1. **Idempotency key** on every business event: consume upstream or generate `payment_request_id` at order time
2. **DB unique constraint** (`ON CONFLICT DO NOTHING`) — the real guard
3. **Manual commit** (`enable.auto.commit=false`, commit after processing completes)
4. **Consider exactly-once** for the Kafka→Kafka hop; use the **outbox pattern** for DB→Kafka
5. **Outbox pattern**: write the event + payment in the SAME DB transaction; a relay publishes the outbox row to Kafka. Only-if-committed events go out
6. **Idempotent consumer processing**: retry/redelivery just hits `ON CONFLICT DO NOTHING`
7. **Monitoring**: Duplicate-rate metric, DLQ for poison paylods that fail validation
8. **Code convention**: all consumers must handle at-least-once; no side effects unless idempotent

## Monitoring

```bash
# Consumer metrics:
# - kafka_consumer_coordinator_commit_total (commit events)
# - kafka_consumer_fetch_manager_records_consumed_total
# - Consumer lag (still matters)
# - Rebalance events & rebalance duration
# - Processing failures per batch

# DB metrics:
# - payments_unique_violation_count (idempotency guard hits)
# - payments_duplicate_detected_total
# - Dead-letter queue size

# Alerts:
# - duplicated event count > 0   → WARNING (should only be benign redelivery)
# - consumer lag > threshold     → WARNING
# - rebalance > 2 per 15 min     → WARNING
```

## Security

- Never log payment_request_id or card data alongside processing details
- Idempotency key must not expose PII or enable key-space guessing
- Exactly-once transactional IDs should be unique per instance; ACLs restrict producer transactional.id
- Sensitive payload validation before persistent write
- Amounts/status integrity — check for negative/overflow; DB constraint on positive amounts

## Production Considerations

- **Reliability**: trust ONLY the database commit; offset commit does not span DB writes unless EOS
- **Scalability**: unique index adds tiny insert overhead; negligible vs correctness
- **Cost**: retries and reprocessing cost compute — idempotency avoids needs for exact repro
- **Compliance**: financial systems MUST be idempotent; audit trail should record uniquely-identified operations
- **Operational**: document the consumer commit semantics in runbooks; demo redelivery behavior in drills
- **HA**: DB unique constraint is the single point of truth across all consumer instances — this is what makes at-least-once safe

## Senior-Level Answer

"Duplicates are inherent to at-least-once semantics with auto-commit; the blind spot is the 5-second window where a processed message hasn't been committed. The correct fix is making processing idempotent: every payment event carries a `payment_request_id`, and the payments table enforces a unique constraint so a redelivered message hits `ON CONFLICT DO NOTHING`. I'd disable auto-commit, commit offsets only after the DB write succeeds, and enable `isolation.level=read_committed` where down-streams exist. For full exactly-once, use the transactional outbox pattern — writing the event and state change in one DB transaction. For Kafka-to-Kafka hops, EOS with transactional producers/consumers gives exactly-once without DB involvement."

## Architect-Level Answer

"The real requirement here isn't just about consumer config — it's a delivery-semantics contract. Payments need idempotent processing because Kafka can't guarantee exactly-once to arbitrary stores. I'd implement the transactional outbox pattern: the payment and its outbox event live in the same transaction, a relay publishes only committed outbox rows to Kafka, and consumers rely on idempotency keys + unique constraints. Where we have Kafka-to-Kafka chains, enable EOS end to end (transactions, read_committed, single transactional.id per producer). I'd also establish a company standard: no business-critical consumer uses auto-commit; and every write path carries a natural idempotency key. Finally, add duplicate-monitoring and DLQ-based alerts so this incident becomes an anomaly, not a class of bug."

## Follow-Up Questions

1. "What data has to be in the idempotency key — and why can't you use Kafka's offset as the key?"
2. "Describe the transactional outbox pattern and how it differs from Kafka exactly-once (EOS)."
3. "If a consumer writes to PostgreSQL AND to an external API, how do you keep those consistent when either can fail?"
4. "What happens to the __consumer_offsets topic and committed offsets when a rebalance occurs mid-processing?"
5. "How do you detect actual duplicate payments in production when idempotency fails, and what's your alerting/corrective flow?"