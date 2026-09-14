# 81. Production Deployment Causes Data Corruption

## Scenario

At 2:00 PM, a deployment was made to production. The deployment included a bug in the data transformation layer that wrote corrupted data to the PostgreSQL database. At 2:30 PM, the corruption was detected — 5,000 transactions have incorrect amounts (multiplied by 1000x). The application has been running for 30 minutes with the corrupted code. The database has Point-in-Time Recovery (PITR) enabled with 5-minute RPO. The corrupted data affects customer balances, transaction amounts, and audit logs. You need to handle rollback, data recovery, reconciliation, and communication.

## Interviewer Question

"A deployment caused data corruption for 30 minutes. 5,000 transactions are affected. How do you handle rollback, data recovery, reconciliation, and communication?"

## What I Should Think About

- Incident severity: P1 (data corruption in banking application)
- Immediate action: stop the bleeding (rollback deployment)
- Data recovery: PITR, WAL replay, transaction logs
- Reconciliation: identify affected records, calculate corrections
- Communication: customers, management, regulators
- Compliance: PCI-DSS, SOX, banking regulations
- Postmortem: root cause, prevention, detection improvement

## Ideal Answer

**Phase 1: Stop the Bleeding (First 5 Minutes)**
- Rollback deployment immediately
- Verify application is running correct code
- Stop any batch jobs that might process corrupted data
- Preserve corrupted data for analysis (don't delete)

**Phase 2: Assess Damage (Next 10 Minutes)**
- Identify all corrupted records
- Calculate scope: 5,000 transactions × $X average = $Y total impact
- Check if corrupted data propagated to other systems (data warehouse, reports)
- Check if any external systems received corrupted data

**Phase 3: Data Recovery (Next 30 Minutes)**
- Use PITR to restore database to 1:55 PM (5 minutes before deployment)
- Replay WAL logs from 1:55 PM to 2:00 PM (5 minutes of good data)
- Create new database with recovered data
- Verify data integrity

**Phase 4: Reconciliation (Next 2 hours)**
- Compare recovered data with corrupted data
- Identify all affected records
- Create correction script for any data that can't be recovered
- Verify all corrections are accurate

**Phase 5: Communication (Ongoing)**
- Notify customers of potential incorrect balances
- Notify management of incident scope
- Notify regulators if required (banking compliance)
- Create incident report for compliance

**Phase 6: Prevention (Post-Incident)**
- Implement pre-deployment data validation
- Add data integrity checks in CI/CD
- Implement canary deployments for data transformation changes
- Add automated rollback on data anomalies

## Architecture

```
┌─────────────────────────────────────────────────┐
│          DATA CORRUPTION INCIDENT                 │
│                                                   │
│  TIMELINE:                                        │
│  1:55 PM  2:00 PM  2:30 PM  3:00 PM  5:00 PM    │
│    │        │        │        │        │          │
│    ▼        ▼        ▼        ▼        ▼          │
│  PITR     Deploy   Detect  Recover  Reconcile    │
│  Point    Bad Code  Issue   DB      Corrections  │
│  (Good)   (Corrupt)        (PITR)   (Validate)   │
│                                                   │
│  RECOVERY PROCESS:                                │
│  ┌──────────────────────────────────────────┐    │
│  │ 1. Rollback deployment                   │    │
│  │ 2. Stop batch jobs                       │    │
│  │ 3. Restore DB from PITR (1:55 PM)       │    │
│  │ 4. Replay WAL (1:55-2:00 PM)            │    │
│  │ 5. Create correction script              │    │
│  │ 6. Apply corrections                     │    │
│  │ 7. Verify data integrity                 │    │
│  │ 8. Communicate to stakeholders           │    │
│  └──────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘
```

## Investigation

1. **Identify corrupted records:**
   ```sql
   -- Find transactions with abnormal amounts
   SELECT id, amount, created_at, status
   FROM transactions
   WHERE created_at >= '2024-01-15 14:00:00'
   AND created_at < '2024-01-15 14:30:00'
   AND (amount > 1000000 OR amount < 0)
   ORDER BY created_at;
   
   -- Count affected records
   SELECT count(*), sum(amount)
   FROM transactions
   WHERE created_at >= '2024-01-15 14:00:00'
   AND created_at < '2024-01-15 14:30:00'
   AND (amount > 1000000 OR amount < 0);
   
   -- Check for data propagation
   SELECT count(*) FROM data_warehouse.transactions
   WHERE source_created_at >= '2024-01-15 14:00:00'
   AND source_created_at < '2024-01-15 14:30:00';
   ```

2. **Check deployment history:**
   ```bash
   # Check deployment at 2:00 PM
   kubectl rollout history deployment/banking-api -n production
   
   # Check deployment details
   kubectl describe deployment/banking-api -n production
   
   # Check for rollback
   kubectl rollout undo deployment/banking-api -n production
   ```

3. **Verify PITR availability:**
   ```bash
   # Check WAL retention
   aws rds describe-db-instances \
     --db-instance-identifier prod-banking-db \
     --query 'DBInstances[0].BackupRetentionPeriod'
   
   # Check latest backup
   aws rds describe-db-instance-backtracks \
     --db-instance-identifier prod-banking-db
   ```

## Commands

```bash
# 1. Rollback deployment immediately
kubectl rollout undo deployment/banking-api -n production

# 2. Stop batch jobs
kubectl patch cronjob batch-processor -n production --type merge -p '{
  "spec": {"suspend": true}
}'

# 3. Create database backup before recovery
aws rds create-db-snapshot \
  --db-instance-identifier prod-banking-db \
  --db-snapshot-identifier pre-recovery-$(date +%Y%m%d%H%M)

# 4. Restore from PITR (point in time recovery)
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier prod-banking-db \
  --target-db-instance-identifier prod-banking-db-recovered \
  --restore-time "2024-01-15T13:55:00Z"

# 5. Create correction script for remaining data
cat > correct_transactions.sql << 'EOF'
-- Correct corrupted transactions
UPDATE transactions
SET amount = amount / 1000,
    updated_at = NOW(),
    correction_note = 'Corrected from data corruption incident 2024-01-15'
WHERE created_at >= '2024-01-15 14:00:00'
AND created_at < '2024-01-15 14:30:00'
AND amount > 1000000;

-- Verify corrections
SELECT id, amount, correction_note
FROM transactions
WHERE correction_note IS NOT NULL
AND created_at >= '2024-01-15 14:00:00';
EOF

# 6. Apply corrections
psql -h prod-banking-db.cluster-xxx.us-east-1.rds.amazonaws.com \
  -U admin -d banking -f correct_transactions.sql

# 7. Verify data integrity
psql -U admin -d banking -c "
  SELECT count(*), sum(amount)
  FROM transactions
  WHERE created_at >= '2024-01-15 14:00:00'
  AND created_at < '2024-01-15 14:30:00';"
```

## Root Cause

| Root Cause | Evidence | Prevention |
|---|---|---|
| Bug in data transformation layer | Amounts multiplied by 1000x | Code review, unit tests |
| No pre-deployment data validation | Corrupted data written | Data validation in CI/CD |
| No canary deployment | Full deployment affected all traffic | Canary deployments |
| No automated rollback on anomalies | 30 minutes of corruption | Automated anomaly detection |
| No data integrity checks | Corruption not detected immediately | Real-time data validation |

## Immediate Mitigation

1. **Rollback deployment** — stop corrupted code
2. **Stop batch jobs** — prevent further corruption
3. **Create database snapshot** — preserve current state for analysis
4. **Restore from PITR** — recover to pre-corruption state
5. **Apply corrections** — fix any remaining corrupted records

## Permanent Fix

1. Implement pre-deployment data validation tests
2. Add canary deployments for data transformation changes
3. Implement real-time data anomaly detection
4. Add automated rollback on data quality metrics
5. Create data integrity monitoring dashboards
6. Implement data reconciliation processes

## Monitoring

- **Data quality metrics** — alert on abnormal amounts, null values
- **Deployment impact** — monitor data quality after each deployment
- **Transaction anomalies** — alert on amounts > 3 standard deviations
- **Data reconciliation** — daily reconciliation between systems
- **Audit log monitoring** — detect unauthorized data changes

## Security

- Corrupted data may contain sensitive information
- Access to recovery process should be restricted
- All corrections should be logged for audit
- Notify compliance team of data corruption incident
- Consider breach notification requirements

## Production Considerations

- **5,000 transactions** — manual review may be required for accuracy
- PITR recovery may take 30-60 minutes for large databases
- Consider using logical replication for faster recovery
- Cost: RDS PITR costs $0.02/GB for backup storage
- Compliance: banking regulations require incident reporting

## Senior-Level Answer

"I'd follow a 6-phase approach: (1) Stop the bleeding — rollback deployment immediately, (2) Assess damage — identify scope of corruption, (3) Data recovery — PITR to pre-corruption state, (4) Reconciliation — identify and correct remaining corrupted records, (5) Communication — notify customers, management, regulators, (6) Prevention — implement data validation, canary deployments, anomaly detection. The key insight is that data corruption requires both technical recovery and business reconciliation."

## Architect-Level Answer

"At the organizational level, I'd establish: (1) Data Integrity Standards — all data transformations must have validation tests, (2) Deployment Safety — canary deployments with automatic rollback on data anomalies, (3) Recovery Architecture — PITR, logical replication, and backup strategies, (4) Compliance Framework — incident reporting, audit trails, data governance, (5) Testing Culture — data quality tests in CI/CD, not just code tests."

## Follow-Up Questions

1. "How do you implement canary deployments for database schema changes?"
2. "What's the difference between PITR and logical replication for data recovery?"
3. "How do you handle data corruption in a distributed system with multiple databases?"
4. "How would you implement real-time data anomaly detection?"
5. "What's your approach to data reconciliation across multiple systems?"
