# 89. Database Backup and Recovery Strategy

## Scenario

Your company's production PostgreSQL database (500GB) currently has a daily snapshot with a 24-hour RPO. The business has grown and now requires:
- RPO: 5 minutes (maximum data loss tolerated)
- RTO: 15 minutes (maximum time to restore service)

The database is transactional (payments, orders, user data). Business leadership wants this validated with an actual DR drill. You currently use a nightly `pg_dump` backup with full + WAL shipping to S3. The database runs on AWS RDS MySQL-compatible... no — it runs on AWS RDS for PostgreSQL (managed). It's in us-east-1 single-AZ (not even multi-AZ). You're asked to design the new backup/recovery strategy that meets the new RPO/RTO and document the DR drill.

## Interviewer Question

"A production database needs a DR strategy. Current backup is a daily snapshot with 24-hour RPO. The business now requires RPO of 5 minutes and RTO of 15 minutes. How do you design the backup and recovery strategy — and how would you prove it works?"

## What I Should Think About

- RPO vs RTO definition and tradeoffs
- Managed vs self-managed backups on RDS/Postgres
- Continuous backups (PITR) vs snapshots: RDS automated backups give PITR to 5 min
- The 5-min RPO requires point-in-time recovery (WAL/PITR), NOT just daily snapshots
- The 15-min RTO: pre-provisioned replica (hot standby) vs restoring from snapshot+WAL
- RDS Multi-AZ for high availability (failover seconds) vs DR (region failover)
- Cross-region replicas for disaster recovery
- Testing/DR drill methodology (restore validation)
- Backup retention policies, compliance, cost

## Ideal Answer

**Analysis:**

Current: daily snapshot → 24h RPO. New: 5 min RPO / 15 min RTO.

For RPO ≤ 5 min, we need continuous backup → point-in-time recovery. In RDS, automated backups with binary log/WAL archiving give PITR to within ~5 min. We should enable:

1. RDS Automated Backups (retention ≥ 7 days) → gives snapshots + transaction logs for PITR
2. Enable enhanced features like multi-AZ for HA on the primary
3. For DR (regional disaster), cross-region read replica OR export to S3 into another region

For RTO ≤ 15 min, restore-from-snapshot of 500GB + replay WAL won't meet 15 min (restore can take 30-60 min for 500GB). So we need either:
- A hot standby (read replica continuously applying WAL) that can be promoted in minutes → meets RTO ~5-10 min
- Or pre-staged restore: schedule dummy restores to warm snapshots

So the recommendation:
- Enable RDS automated backups (snapshots + WAL → 5-min RPO)
- Enable a cross-region read replica in the DR region (async replication → near-zero to 5-min RPO; failover = promote replica)
- Validate with a DR drill each quarter
- Use Multi-AZ on the read replica as well (or rely on RDS)
- Optionally, replicate snapshots to DR region

**Decision tables:**

| Approach | RPO | RTO | Cost | Complexity |
|---|---|---|---|---|
| Daily snapshot only | 24h | hours | low | low |
| RDS auto backup (PITR) | ~5min | 30-90min restore | medium | low |
| RDS PITR + read replica (DR) | ~5min (async) | 5-15 min (promotion) | high | medium |
| Logical dump (pg_dump) nightly | 24h | hours | low | medium |

→ Choose RDS PITR + read replica in DR region to meet targets.

## Architecture

```
  PRODUCTION REGION: us-east-1
  ┌─────────────────────────────────────────┐
  │  VPC A                                   │
  │  ┌──────────────────────────────┐       │
  │  │ AZ1 Primary (multi-AZ)       │       │
  │  │ PostgreSQL (r6g.4xlarge)     │       │
  │  │ 500GB                         │       │
  │  │ ── synchronous standby in AZ2│       │
  │  └──────┬───────────────────────┘       │
  │         │                              │
  │         │ WAL streaming (continuous)   │
  │         │ + S3 automated backups       │
  │         ▼                              │
  │  ┌─────────────────────────────┐       │
  │  │ S3 bucket: prod-db-backups   │       │
  │  │  snapshots (daily)           │       │
  │  │  wal files (continuous→PITR) │       │
  │  └─────────────────────────────┘       │
  └───────────────┬────────────────────────┘
                  │ cross-region replication
                  │ (WAL → DR replica) + snapshot copy
                  ▼
  DR REGION: us-west-2
  ┌─────────────────────────────────────────┐
  │  VPC B                                   │
  │  ┌──────────────────────────────┐       │
  │  │ Read replica (r6g.4xlarge)    │       │
  │  │ continuously applying WAL     │       │
  │  │ Async lag: ~1-5s               │       │
  │  │ PROMOTABLE → new primary      │       │
  │  └──────────────────────────────┘       │
  │                                         │
  │  DR RUNBOOK:                            │
  │  1. Detect region failure (Route53)     │
  │  2. Promote replica (RDS failover)     │
  │  3. Redirect DNS/app connections        │
  │  4. ~5-15 min to RTO                    │
  └─────────────────────────────────────────┘

  BACKUP SCHEDULE:
  ┌─────────────────────────────────┐
  │ RDS automated backups:           │
  │  retention: 7-35 days            │
  │  snapshot: daily 02:00 UTC       │
  │  WAL upload: continuous (5min RPO)│
  ├─────────────────────────────────┤
  │ Logical backup (compliance/audit):│
  │  pg_dump nightly to S3           │
  │  (for corruption recovery,       │
  │   independent of RDS storage)    │
  └─────────────────────────────────┘
```

## Investigation

**Step 1: Assess current state**
```bash
# Check current backup configuration
aws rds describe-db-instances --db-instance-identifier prod-db \
  --query 'DBInstances[0].{AZ:DBInstanceClass,BackupRetention:BackupRetentionPeriod,MultiAZ:MultiAZ,StorageEncrypted:StorageEncrypted,Engine:Engine}'

# Check if PITR (automated backups) is enabled
aws rds describe-db-instances --db-instance-identifier prod-db \
  --query 'DBInstances[0].BackupRetentionPeriod'
```

**Step 2: Check existing backups and size**
```bash
aws rds describe-db-snapshots --db-instance-identifier prod-db \
  --query 'DBSnapshots[].{Id:DBSnapshotIdentifier,Created:SnapshotCreateTime,Status:Status}'

# Estimate restore time
# ~30-60 min for 500GB snapshot restore
```

**Step 3: Validate current RPO/RTO with a restore drill**
```bash
# Restore a snapshot to a new instance (measure time)
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier dr-restore-test \
  --db-snapshot-identifier <snapshot-id>

# Time until AVAILABLE → measure
# Replay WAL to a point in time
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier prod-db \
  --target-db-instance-identifier pitr-test \
  --restore-time '2026-09-14T12:15:00Z'
```

## Commands

```bash
# ENABLE automated backups (PITR)
aws rds modify-db-instance \
  --db-instance-identifier prod-db \
  --backup-retention-period 7 \
  --preferred-backup-window 01:00-02:00 \
  --preferred-maintenance-window sun:04:00-sun:05:00 \
  --apply-immediately

# CREATE cross-region read replica (DR)
aws rds create-db-instance-read-replica \
  --db-instance-identifier prod-db-dr-replica \
  --source-db-instance-identifier prod-db \
  --db-instance-class db.r6g.4xlarge \
  --region us-west-2 \
  --availability-zone us-west-2a \
  --multi-az \
  --publicly-accessible false

# PROMOTE replica for DR (failover)
aws rds promote-read-replica \
  --db-instance-identifier prod-db-dr-replica

# MANUAL snapshot before major changes
aws rds create-db-snapshot \
  --db-instance-identifier prod-db \
  --db-snapshot-identifier prod-db-pre-deploy-$(date +%Y%m%d)

# RESTORE from snapshot (RTO test)
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier dr-test \
  --db-snapshot-identifier prod-db-snapshot \
  --db-instance-class db.r6g.4xlarge

# Restore to PITR (within 5 min granularity)
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier prod-db \
  --target-db-instance-identifier pitr-recovery \
  --restore-time 2026-09-14T08:30:00Z

# LOGICAL backup for independence (pg_dump)
pg_dump -h prod-db.xyz.us-east-1.rds.amazonaws.com \
  -U backup_user -Fc prod \
  | aws s3 cp - s3://backups/prod/daily-$(date +%Y%m%d).dump

# Copy snapshot to DR region
aws rds copy-db-snapshot \
  --source-db-snapshot-identifier arn:aws:rds:us-east-1:123:snapshot:prod-snap \
  --target-db-snapshot-identifier prod-snap-dr \
  --source-region us-east-1 \
  --region us-west-2

# Restore from DR-region snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier dr-recovered \
  --db-snapshot-identifier prod-snap-dr \
  --region us-west-2
```

## Root Cause

This isn't a "root cause" incident per se — it's a gap analysis. The business requirement (RPO 5 min, RTO 15 min) doesn't match the current implementation:
1. Daily snapshot → RPO 24h (violates 5-min requirement)
2. Snapshot restore takes too long for 500GB → RTO > 15 min (violates)
3. No WAL/PITR enabled → can't recover to within minutes
4. No DR environment → in a region failure, no recovery at all

## Immediate Mitigation

```bash
# 1. ENABLE PITR immediately (automated backups + WAL)
aws rds modify-db-instance \
  --db-instance-identifier prod-db \
  --backup-retention-period 7 --apply-immediately

# 2. Create DR read replica in second region
aws rds create-db-instance-read-replica \
  --db-instance-identifier prod-dr \
  --source-db-instance-identifier prod-db \
  --region us-west-2

# 3. Take manual snapshot (baseline)
aws rds create-db-snapshot \
  --db-instance-identifier prod-db \
  --db-snapshot-identifier prod-baseline-$(date +%Y%m%d%H%M)

# 4. Enable Multi-AZ on primary for HA
aws rds modify-db-instance \
  --db-instance-identifier prod-db \
  --multi-az --apply-immediately
```

## Permanent Fix

1. **Enable continuous backup (PITR)** honoring 5-min RPO
2. **Pre-provision a hot standby** (read replica + RDS failover) for 15-min RTO
3. **Automated DR drill script** (monthly/quarterly):
   - Test snapshot restore time
   - Test PITR restore to a specific timestamp
   - Test replica promotion + app connection switch
4. **Documented runbook** for failover and failback
5. **Data validation checks** after restore (row counts, checksums vs app-level)
6. **Retention policy** aligned with compliance (7-35 days automated; archive snapshot monthly)
7. **Cost optimization**: schedule DR drill restores to run only when needed; right-size the DR replica

## Monitoring

```bash
# CloudWatch alarms:
# - SnapshotCreateFailure
# - BackupTaskFailure
# - ReplicaLag (DR replica)
# - CPUUtilization on DR replica
# - FreeStorageSpace (correlated to WAL growth)

# RDS events:
aws rds describe-events --source-type db-instance \
  --source-identifier prod-db --duration 1440

# Backup success monitoring:
# - Verify daily snapshot creation
# - Verify WAL upload continuing (S3 object count growing)
# - DR drill results recorded in dashboard

# Dashboard:
# - Last successful snapshot (age)
# - Estimated RPO (snapshot age + WAL gap)
# - Estimated RTO from last drill measurement
# - DR replica lag
```

## Security

- Encrypt backups at rest (KMS) and in transit
- DR replica in private subnet, no public access
- Backup bucket versioning and lifecycle (compliance)
- Restrict access to backup bucket via IAM policies
- Backup credentials: separate role, no long-lived keys
- Validate encrypted restore (KMS keys must be available in DR region — copy keys or use multi-region key policy)
- Audit who performs restore/promote operations

## Production Considerations

- **HA vs DR**: Multi-AZ (same region) handles AZ loss; cross-region replica handles region loss. Both needed
- **RTO**: 15-min RTO hinges on pre-provisioned replica. If budget doesn't allow, explore pausing replica (stop/start) to cut cost while keeping snapshot of DR
- **Cost**: continuous WAL and replicas are more expensive than daily snapshots; justify to leadership with RPO/RTO value
- **Compliance**: PCI/SOC2 requires documented backup + recovery with evidence (drill reports)
- **Operational**: DR runbook must be tested with actual app; include DNS/DNS failover (Route53 `failover` routing policy)
- **RPO edge cases**: logical replication vs streaming — streaming lag is seconds, so RPO 5 min is easily met on replica; on PITR, WAL upload is near-continuous
- **Failback**: after DR, build new replica in DR region, promote primary back, and re-establish replication

## Senior-Level Answer

"I'd enable RDS automated backups with 7-day retention (continuous WAL → 5-min RPO) and create a cross-region read replica in the DR region that provides both hot standby (failover in minutes → 15-min RTO) and near-zero RPO via async streaming. Then validate with a quarterly DR drill that measures actual restore/promote times. I'd also keep a periodic logical `pg_dump` as an independent recovery option and monitor snapshot success, WAL upload, replica lag, and drill results. Every promotion is rehearsed so the team isn't figuring it out during a real outage."

## Architect-Level Answer

"This becomes a resilience architecture decision: we need tiered protection. We can't achieve 15-min RTO with cold restoration of 500GB, so the DR strategy must be active standby: a continuously-applyng read replica (RPO seconds) that can be promoted (RTO minutes). I'd define the failover as part of a broader regional disaster runbook that includes DNS failover (Route53 health-check-based), app connection pooling that reconnects to the new primary, and cross-region replica promotion automation. I'd also add, at the application level, an outbox pattern so transactional data changes are captured — protecting against logical corruption that replication can't fix. And I'd implement digest: restore drills, RPO/RTO dashboards, and a data validation suite. Multi-AZ in the primary region covers AZ failure; the cross-region replica covers region failure — that layered design is what actually meets the business targets."

## Follow-Up Questions

1. "If you can't afford keeping a DR replica running 24/7, what alternatives meet a 15-minute RTO?"
2. "How do you validate that a PITR restore actually has consistent, usable data?"
3. "Explain the difference between RDS Multi-AZ failover and promote-read-replica coping with RPO/RTO."
4. "How would you handle a logical corruption (accidental table DROP) when a physical snapshot is also corrupted?"
5. "What are the tradeoffs of logical replication vs physical streaming replication for DR in this design?"