# 100. Complete Data Center Failure - Disaster Recovery Activation

## Scenario

The primary data center (Region A) is completely down due to a power failure. The application serves 10 million users and processes 50,000 transactions per second (TPS). This is a payments platform. You have a DR site in Region B (another AWS region, 800ms away). It has:
- A cross-region read replica of the primary database (RDS PostgreSQL), ~5 seconds of lag
- A standby application cluster deployed and configured (Docker/Kubernetes images ready, scaled to zero)
- Cache, CDN, message brokers (Kafka) configured in DR with empty clusters
- Route53 health checks on the primary ALB

It's 09:00 on a Tuesday morning. Traffic is at its daily peak. You're the DevOps/architect lead. Leadership asks: "Activate the DR site. When will we be back up, and with how much data loss?"

## Interviewer Question

"The primary data center is completely down. You have a DR site in another region: a standby cluster, cross-region DB replica, and empty Kafka. Design the complete failover process, validate the DR environment, handle failback, and discuss RTO, RPO, active-passive vs active-active. Include the architecture."

## What I Should Think About

- Active-passive (warm standby) vs active-active (multi-region active) — this scenario is warm standby
- RPO here is governed by cross-region DB replication lag (~5s) + Kafka replication gap → potential loss window
- RTO is governed by: detection, promotion/extraction of DB, app bootstrap, DNS cutover, cache warm, Kafka replay
- Failover sequence (ordered dependency graph): DB promote → database availability → app scale → Kafka rebuild → DNS → cache → monitoring/verification
- Validation of DR: runbook checks, smoke tests, real traffic canary
- Data loss handling: reconcile post-failover (replay lost WAL/messages, manual reconciliation)
- Safety: block writes to primary if it comes back split-brain; fencing
- Failback: catch-up of Region A once it's back (reverse replication), switch back with zero/minimal loss
- DNS/TTL: lower TTL before incident; GRPOC health checks; consider PrivateLink/ALB

## Ideal Answer

Because primary is fully down, we cannot survive on it — failover is mandatory. This is a **warm-standby (active-passive)** DR model. RTO budget must fit within business target. Let's assume business SLA: RTO 30 min, RPO ≤ 5 min.

**Sequence (with approximate RTO pixels):**

1. `T+0min` Confirm outage; declare DR activation; take evidence
2. `T+2min` Route53 health check fails over; but nothing's actually running yet — we run the failover RUNBOOK, not just DNS
3. `T+5min` Promote cross-region DB replica (replica has ~5s lag → RPO ≈ 5s + any in-flight). Use `aws rds promote-read-replica`. Then enable writes, update the app connection strings (SecretsManager/RDS)
4. `T+10min` Scale the DR application cluster: `kubectl scale` deployments from 0, OR use HPA autoscaled. Bring up services. Wait health.
5. `T+15min` Signal Kafka: the DR Kafka cluster from the primary is empty — messages that were in the primary's brokers are LOST (RPO exposure). We must replay retained offsets or reprocess from a durable datastore. If no durable source → data loss window = # messages in Kafka at failure.
6. `T+20min` Verify: run smoke tests (health probes, log in, process a test payment), add a canary route; when green → full DNS cutover to DR ALB.
7. `T+25-30min` Full traffic to DR. Monitor error rate and lag.

**Data loss validation:** We lost everything in Kafka and any primary DB transactions not yet replicated. In-flight money-movement events must be reconciled. Post-failover reconciliation job: compare DR DB vs any durable event source (outbox table, payment processor ledger) — produce the "last reconciled point".

## Architecture

```
  NORMAL (ACTIVE-PASSIVE):
  Region A (PRIMARY)                    Region B (DR / warm standby)
  ┌──────────────────────────────┐      ┌───────────────────────────────┐
  │ CDN/CloudFront               │      │ CDN/CloudFront                 │
  │ ALB -> pods (active)         │      │ ALB -> pods (scaled 0)  ----  │
  │ app cluster (200 replicas)   │      │  standby app images set      │
  │ primary DB (RDS, writes)     │      │  ┌───────────────────────┐   │
  │ Kafka cluster (producers)    │      │  │ DR read replica (lag ~│   │
  │ Redis cache                  │      │  │  5s)                  │   │
  └───────────────┬──────────────┘      │  │ Kafka DR (empty)      │   │
                  │ replication (async) │  │ Redis DR (empty)      │   │
                  ├────────────────────▶│  └───────────▲───────────┘   │
                  │ (WAL → RDS replica, │              │                │
                  │  cross-region)      │              │ (failover ->   │
                  └─────────────────────┘              │  promote)      │
                                                        └───────────────┘

  AFTER FAILOVER:
  Region A (DOWN)                 Region B (NOW PRIMARY - ACTIVE)
  ┌─────────────────────┐         ┌──────────────────────────────┐
  │ black/power failure │         │ ALB → pods scaled up (active)│
  │ everything offline  │         │ promoted DB (writes ON)      │
  │  (no more producer  │  ──────▶│ Kafka DR rebuilt (replayed)  │
  │   writes; 728 MSEs) │  Route53│ Redis DR filled from DB      │
  └─────────────────────┘  switch │ CDN switch                    │
                                 └──────────────────────────────┘

  FAILBACK (Region A returns):
  Region A (catch-up)                  Region B (still primary)
  ├── reboot infra
  ├── rebuild DB from Region B via WAL
  ├── configure reverse replication
  ├── run consistency checks
  └── switch writes back at low-traffic window (planned cutover)
```

## Investigation

**Step 1: Declare and confirm full outage**
```bash
# Confirm readiness of both regions
aws elbv2 describe-load-balancers --region us-east-1 --query 'LoadBalancers[].DNSName'   # Region A
curl -I https://api.example.com/health   # Region A → fail
# Confirm Route53 health check state
aws route53 get-health-check-status --health-check-id <id> --region us-east-1
# Confirm Region B is healthy but scaled to 0
kubectl get nodes --context dr
kubectl get deploy -n prod --context dr
```

**Step 2: Validate region B infra before cutover**
```bash
kubectl get pods -n prod --context dr | grep -c Running
aws rds describe-db-instances --region us-west-2 \
  --query 'DBInstances[].{Status:DBInstanceStatus,Replica:ReadReplicaSourceDBInstanceIdentifier}'
aws ec2 describe-subnets --region us-west-2
```

**Step 3: Check cross-region DB lag at failover time**
```sql
-- BEFORE promotion (RPO forensics): measure lost data
-- On DR replica:
SELECT now() - pg_last_xact_replay_timestamp() AS lag_seconds;
-- Record last LSN: RPO checkpoint for reconciliation
SELECT last_committed_lsn FROM pg_stat_replication; -- or replay_lsn
```

**Step 4: Identify what was in Kafka at the moment of failure**
```bash
# The primary's Kafka cluster is unreachable → we can't read its offsets directly.
# Use the durable outbox/ledger to reconstruct the message set:
SELECT state, MAX(id) max_event_id, COUNT(*) FROM outbox_events GROUP BY state;
# Compare with what DR consumers have processed.
# Estimate lost messages ≈ (consumer offset in DR Kafka) vs (outbox last id).
```

## Commands (Failover Runbook Snippet)

```bash
# STEP 1: Promote DR DB replica (RPO ~5s)
aws rds promote-read-replica \
  --db-instance-identifier dr-prod-replica \
  --region us-west-2

# wait until STATUS=available, then enable writes:
# (RDS read replica promote becomes a primary automatically)

# STEP 2: Point the app to the promoted DB
#    (from SecretsManager/ConfigMap; update connection string)
aws secretsmanager update-secret \
  --secret-id prod/api/db --secret-string '{"host":"dr-prod.xyz.us-west-2.rds.amazonaws.com",...}'

# STEP 3: Scale the DR application cluster up
kubectl scale deploy api --replicas=150 --context dr -n prod
kubectl rollout status deploy/api --timeout=300s --context dr -n prod

# STEP 4: Kafka recover + replay:
#   - Start DR Kafka brokers
#   - Configure MirrorMaker 2 to read the (unreachable) primary? No —
#     instead: re-drive events from the durable outbox table into DR Kafka:
#       (1) drain outbox_events to kafka DR topic (the ledger, not lost)
#       (2) set consumer offsets to align with replayed window
psql -h dr-prod... -U app -c \
  "SELECT id, topic, payload FROM outbox_events WHERE published=false ORDER BY id"

# STEP 5: DNS cutover (Route53 failover record)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123 --change-batch file://failover-dns.json
# records: api.example.com → DR ALB (health request -> 200)

# STEP 6: Cache: fill Redis from app reads; no manual action except warm start
# STEP 7: Verify + canary
curl -s https://api.example.com/health -m 5
kubectl -n prod --context dr get hpa,deploy,svc
```

## Immediate Mitigation

```bash
# 1. Freeze writes to Region A intent (nothing to write; it's dead) — 
#    but add a safety re: split-brain: any Region A resources that COME BACK
#    must NOT serve writes while Region B is primary (fencing).
#    → Change Region A to maintain read replica but no writes (promotion).

# 2. Promote DB + app + Kafka (as above) with dry-run in staging first.

# 3. Block Region A from rejoining as primary (set as read-only / disabled).
aws rds modify-db-instance --db-instance-identifier prod-db \
  --region us-east-1 --backup-retention-period 0   # (or pause)

# 4. Do FULL traffic cutover only after smoke tests pass.
# 5. Post-failover reconciliation job runs continuously:
#     - compare DR DB with ledger/outbox
#     - identify data-loss window precisely
#     - produce "lost events" report for business/finance team.
```

## Permanent Fix

1. **Move toward active-active (dual-write or partition by customer segment)** for the true 0-RTO; otherwise keep warm standby but (a) pre-scale DR higher, (b) automate failover orchestration (Terraform + a DR orchestrator), (c) run periodic DR drills (quarterly) with measured RTO/RPO
2. **Reduce RPO:**
   - Synchronous DB replication for critical writes (regional expansion cost)
   - Duel-durable outbox (apply WAL to DR) so no Kafka/data loss
   - Kafka MirrorMaker 2 cross-region (async) to DR Kafka → keeps a live copy in DR
3. **Reduce RTO:**
   - Pre-scale DR app min-replicas (autoscaling ready within 5 min)
   - Automated failover orchestration (Lambda/Step Functions) — runbook-as-code
   - Pref-tested DNS TTL (lower to 60s), PrivateLink/ALB warm
4. **Failback plan:** when Region A returns, do planned cutover: replicate Region B → Region A, verify, switch at low traffic; keep Region B live until cutover verified
5. **Testing**: quarterly Chaos/DR drill: kill Region A in staging, measure the RTO/RPO certified

## Monitoring

```bash
# Region/DR monitoring:
# - Cross-region replication lag (DB, Kafka MirrorMaker) → alert at >10s
# - Region health synthetic probes (availability target)
# - App health sync: all secret connections point to PRIMARY ROLE
# - RTO/RPO metrics recorded on each drill/failover
# - Split-brain detector: Region A "should it still serve writes?" signal

# Alerts:
# - DB replication lag > 10s → page
# - Kafka MirrorMaker lag > threshold → page
# - Any Region A writes while B is primary → CRITICAL (fencing broken)
# - DR drill not run in 90 days → compliance alert

# Dashboards: real-time status of primary vs DR, RPO estimate,
# failover state machine state, reconciliation gap.
```

## Security

- Crossing regions: encrypt data in transit (TLS) + at rest (KMS, with cross-region KMS key for the replica)
- No cross-region plaintext; secrets (DB creds for DR) stored in DR Vault/SM — validate before failover
- Post-failover: rotate/audit service credentials used in cutover
- Access control: DR failover should require break-glass approval (multi-party) — guard with IAM policies + audit trail
- Compliance: payments data in DR region → ensure residency/perimeter compliance before moving traffic (data sovereignty)

## Production Considerations

- **HA**: active-passive gives RTO minutes; active-active gives near-zero RTO but costs double and adds consistency complexity — size to the business requirement
- **Scalability**: DR app pre-scaled; HPA on the DR region; DB promoted instance sizing matches primary
- **Reliability**: failover order dependency (DB → app → Kafka → DNS) must be scripted to avoid partial states
- **Cost**: warm standby costs money when idle (2nd region infra). Consider "pause replica + scale-to-0" in off-hours, but test restart
- **Compliance**: financial data + failover = evidence: post-incident RPO report, preservation of audit trails, notification to regulators if outage affects SLA
- **Operational**: DR runbook owned by an authored team with approval workflow; every quarter a rehearsal
- **RTO/RPO**: stated contract > measured reality — if holiday DR target is 30min, measure it in a drill, then tune (pre-scale orders, automation) until met

## Senior-Level Answer

"Since primary is down completely, this is a determined warm-standby failover: promote the DR replica (~5s lag → ~5s RPO), scale up the DR app from zero, rebuild Kafka from the durable transaction ledger, then cut DNS. RTO realistically 25-40 minutes including promotion, bootstrap and verification — I'd pre-scale and pre-automate to hit 15. Key risk is Kafka: whatever was in the primary broker is gone; we must reconcile from the outbox/ledger and quantify the loss to the business. Post-failover, I'd add fencing so a recovered Region A can't serve writes (split-brain), then run continuous reconciliation. Failback is a planned cutover at low traffic if we convert Region B to replicate back. Active-active would need dual-write/partitioning designs — different tradeoffs, worth evaluating."

## Architect-Level Answer

"This incident reveals the true DR model we operate: warm-standby means 'acceptable known data-loss window'. We have the RPO (5s replication lag + Kafka in-flight) and RTO (25-40 min) as explicit business expectations. Strategically, for a 50k TPS payments platform, I'd push toward active-active: partition traffic by customer segment across two regions (dual-region writes with CRDT/outbox-based reconciliation), giving RPO 0 and RTO seconds for the carrying side. Pragmatically, we can increment: (1) add MirrorMaker 2 and a cross-region durable outbox to eliminate Kafka loss; (2) implement failover-as-code (Step Functions/Terraform) with pre-scaled DR; (3) quarterly chaos drills measuring certified RTO/RPO. Failback is a planned path with reverse replication + verification. Encode all of this as an architecture gate: any new service ships with a defined DR+replication story."

## Follow-Up Questions

1. "If active-active meant synchronous replication across regions with money to spend, on which components would you spend first and why?"
2. "How do you design the reconciliation job that identifies exactly which transactions were lost in the 5-second + Kafka gap?"
3. "Describe the fencing mechanism for a region-intented failback — how do you guarantee no split-brain writes?"
4. "What's the role of the shift: promote DR replica as READ-WRITE vs making the primary read-only — and how do you sequence service turn-up from zero safely?"
5. "Design a DR drill that measures RTO/RPO without affecting the real production traffic — what do you simulate and how do you attribute?"