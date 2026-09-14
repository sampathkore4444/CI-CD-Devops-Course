# 32. Rolling Back a Failed Deployment

## Scenario

At 9:00 AM, you deployed version 3.5.0 of your e-commerce platform to production. The deployment completed successfully — all pods came up healthy, health checks passed, and the canary looked good for 10 minutes before you promoted to full rollout. By 10:30 AM, customer support reports are flooding in: approximately 5% of transactions are being corrupted. The bug is in the new price calculation logic — it's applying a currency conversion twice for international orders, resulting in double-charged customers. The payment processor is processing these transactions normally (the amounts are just wrong). You need to rollback immediately, but you have concerns: the database had a migration applied during deployment, some customers have already been charged incorrect amounts, and the new API version is being consumed by the mobile app (version 4.2.1) which expects the new response format. The rollback is not simple.

## Interviewer Question

"Your deployment succeeded but there's a data corruption bug affecting 5% of transactions. Walk me through the complete rollback process — not just the application, but the database, configuration, client apps, and communication. What's your runbook?"

## What I Should Think About

- This is a multi-layered rollback: application, database, configuration, and client
- 5% data corruption means some customers were charged incorrect amounts — financial impact
- The database migration adds complexity — can we safely rollback the schema?
- The mobile app depends on the new API format — rollback may break the mobile client
- Need to communicate to customers, support team, and stakeholders
- Must preserve evidence for post-incident review
- Think about whether a forward fix is better than a rollback
- Consider the order of rollback steps (what depends on what)

## Ideal Answer

**Immediate Actions (First 5 minutes)**

1. **Declare incident**: Page the incident commander, open a war room
2. **Rollback the application**: Revert to the last known good version (3.4.2)
3. **Halt the bleeding**: If possible, disable the affected feature via feature flag

**Application Rollback (5-15 minutes)**

```bash
# Option 1: Kubernetes rollback
kubectl rollout undo deployment/ecommerce-api -n production
kubectl rollout undo deployment/ecommerce-api -n production --to-revision=8

# Option 2: ArgoCD rollback
argocd app rollback ecommerce-api

# Verify rollback
kubectl rollout status deployment/ecommerce-api -n production
kubectl get pods -n production -l app=ecommerce-api -o jsonpath='{.items[*].spec.containers[*].image}'
```

**Database Rollback Decision**

This is the hardest part. Options:

1. **Forward fix**: If the migration is additive (new column, new table), it's safe to leave and rollback only the application code
2. **Backward-compatible rollback**: If the migration modified existing columns, you need a backward-compatible migration that works with both old and new app versions
3. **Full database rollback**: Restore from backup taken before deployment — only if data loss is acceptable for the last 2 hours

```sql
-- If migration was additive (safe to leave):
-- No database rollback needed, just roll back the app

-- If migration modified columns (need backward-compatible fix):
-- Write a "fix-back" migration that's compatible with old app version
BEGIN;
UPDATE orders SET price = price / exchange_rate 
WHERE created_at > '2026-09-14 09:00:00' AND is_international = true;
COMMIT;

-- If full rollback needed (last resort):
-- Restore from pre-deployment snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier ecommerce-prod-backup
```

**Financial Remediation**

1. Generate a report of all affected transactions
2. Issue refunds or credits for double-charged amounts
3. Notify affected customers

**Client Communication**

1. **Support team**: Provide a script of what to tell affected customers
2. **Mobile team**: If the mobile app expects new API format, either:
   - Deploy a backward-compatible API version that serves both formats
   - Push a mobile app hotfix (not ideal, takes days)
   - Use API versioning to serve old format from rolled-back backend
3. **Status page**: Update public status page

## Architecture

```
ROLLBACK DECISION TREE:
┌─────────────────────────────────────┐
│   Deployment Failed in Production   │
└──────────────┬──────────────────────┘
               │
        ┌──────┴──────┐
        │ Application │
        │  Rollback   │ ← Always do this first
        └──────┬──────┘
               │
        ┌──────┴──────┐
        │   Database  │
        │  Rollback?  │
        └──────┬──────┘
               │
    ┌──────────┼──────────┐
    │          │          │
┌───┴───┐ ┌───┴───┐ ┌───┴───┐
│Additive│ │Modify │ │Breaking│
│Migration│ │Schema │ │Change │
│(No fix)│ │(Fix)  │ │(Restore)│
└───────┘ └───────┘ └───────┘
    │          │          │
    ▼          ▼          ▼
 Leave DB   Write      Restore
            Backward   from
            Compat     Snapshot
            Migration

ROLLBACK ORDER:
1. Feature Flag → Disable affected feature (0 sec)
2. Application Rollback → Revert pods (2 min)
3. Database Fix → If needed (5-30 min)
4. Client Communication → Notify users (15 min)
5. Financial Remediation → Refund affected (1-24 hr)
```

## Investigation

1. **Identify the scope**: How many transactions are affected? Query the database for orders since deployment time with incorrect amounts
2. **Determine the rollback scope**: Is it app-only, or app + database?
3. **Check for dependent services**: What other services call the price calculation API?
4. **Review the database migration**: Was it additive or destructive?
5. **Check client applications**: Which clients consume the API and do they depend on the new format?
6. **Assess data corruption extent**: Generate a report of affected transactions
7. **Verify rollback succeeded**: Confirm old version is running and new transactions are correct
8. **Monitor for cascading effects**: Check downstream services for issues caused by incorrect prices

## Commands

```bash
# Immediate application rollback
kubectl rollout undo deployment/ecommerce-api -n production
kubectl rollout status deployment/ecommerce-api -n production

# Verify all pods are on old version
kubectl get pods -n production -l app=ecommerce-api \
  -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image

# Check rollout history
kubectl rollout history deployment/ecommerce-api -n production

# Generate affected transaction report
kubectl exec -it postgres-pod -n production -- psql -c "
SELECT order_id, customer_id, amount_charged, amount_expected, 
       (amount_charged - amount_expected) as overcharge
FROM orders 
WHERE created_at >= '2026-09-14 09:00:00' 
  AND is_international = true
  AND currency != 'USD'
ORDER BY created_at DESC;
"

# Check for dependent services
kubectl get svc -n production | grep -i price
kubectl get virtualservice -n production -o yaml | grep -A5 price-service

# Monitor transaction success rate after rollback
kubectl logs -f deployment/ecommerce-api -n production | grep -i "transaction"
```

## Root Cause

| Root Cause | Mitigation |
|---|---|
| Insufficient integration testing with international orders | Add production-data-shaped integration tests |
| No feature flag for risky code paths | Implement feature flags for all new financial logic |
| Database migration applied before app rollback | Use expand-contract migration pattern |
| No pre-deployment data validation | Add automated checks comparing pre/post deployment metrics |
| Canary didn't catch it (only 5% of traffic) | Canary test suite must include edge cases (international orders) |
| No automated rollback trigger on error rate | Implement automatic rollback based on error rate threshold |

## Immediate Mitigation

1. **Rollback the application immediately** — do not wait for root cause analysis
2. **Feature flag the affected code path** if rollback isn't possible immediately
3. **Pause all payment processing** if corruption is severe (only if business allows)
4. **Notify the incident commander** and open a war room
5. **Generate a report of affected transactions** for financial remediation
6. **Communicate to support team** with a customer-facing script

## Permanent Fix

1. **Implement automatic rollback triggers**: If error rate exceeds threshold (e.g., 1%), automatically rollback
2. **Expand canary test coverage**: Include edge cases like international orders, different currencies
3. **Feature flags for financial logic**: Never deploy financial code without a kill switch
4. **Expand-contract database migrations**: Make migrations backward-compatible so rollback is safe
5. **Pre-deployment snapshot**: Always snapshot the database before deploying migrations
6. **Transaction validation**: Add real-time transaction amount validation as a safety net
7. **Post-deployment monitoring**: Automated comparison of pre/post deployment transaction patterns

## Monitoring

- **Transaction amount distribution**: Alert if average transaction amount changes significantly post-deployment
- **Error rate on payment processing**: Alert if > 0.5% of transactions fail
- **International order success rate**: Separate metric for edge cases
- **Rollback success confirmation**: Alert if pod versions don't match expected after rollback
- **Financial reconciliation**: Automated daily check comparing charged amounts vs. expected amounts
- **Deployment markers**: Annotate metrics with deployment version for correlation

## Security

- Financial data corruption may have compliance implications (PCI DSS)
- Audit logs for all refund/credit transactions
- Ensure rollback doesn't expose old vulnerabilities
- Customer PII in transaction reports must be handled according to GDPR/CCPA
- Database backups must be encrypted and access-controlled
- Incident response must preserve evidence for compliance audits

## Production Considerations

- **Reliability**: Rollback time should be under 5 minutes for financial systems
- **Scalability**: Rollback procedures should work across multiple replicas
- **Cost**: Database snapshots cost money but are essential for rollback capability
- **Compliance**: Financial transaction corrections must be auditable
- **Operational**: Rollback runbooks must be tested quarterly
- **Customer Trust**: Transparent communication about data issues builds trust

## Senior-Level Answer

"I'd execute a layered rollback: first, roll back the application using `kubectl rollout undo` to restore the previous version. For the database, if the migration was additive (new column), I'd leave it and just roll back the app code. If it modified existing data, I'd write a backward-compatible fix migration. I'd immediately generate a report of affected transactions for financial remediation, communicate to the support team with a customer script, and coordinate with the mobile team about API compatibility. The key insight is that the rollback order matters — app first (stops the bleeding), then database if needed, then remediation. I'd also implement automatic rollback triggers based on error rate thresholds so this happens faster in the future."

## Architect-Level Answer

"This incident exposes the need for a comprehensive deployment safety architecture. I'd implement: **Feature flags** for all financial logic, allowing instant disabling without deployment. **Expand-contract database migrations** that are always backward-compatible, making rollbacks safe. **Automatic rollback controllers** that monitor error rates post-deployment and trigger rollback if thresholds are breached. **Transaction validation microservice** that sits as a safety net before payment processing, catching anomalous amounts. **Deployment orchestration with staged rollout** — canary → 10% → 50% → 100% with automated gates at each stage. The architecture should make it so that rolling back is a routine, automated operation rather than a crisis response. At the enterprise level, I'd establish a **Deployment Safety Standard** across all teams with mandatory feature flags for high-risk changes, backward-compatible migrations, and automated rollback capabilities."

## Follow-Up Questions

1. "Your rollback succeeded but the mobile app (version 4.2.1) still expects the new API format. How do you handle the client-side compatibility?"
2. "How would you implement automatic rollback based on error rate thresholds in Kubernetes?"
3. "What if the database migration was destructive (dropped a column)? How does that change the rollback strategy?"
4. "How do you run a post-incident review that actually leads to systemic improvements?"
5. "How would you design a deployment pipeline that makes this class of failure architecturally impossible?"
