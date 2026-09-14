# 110. Deployment Succeeds but Business Transactions Fail

## Scenario

The deployment pipeline shows all green. Health checks pass. The application starts and responds to HTTP requests. But business transactions are failing silently. Orders are created but never processed. Payments are deducted but orders are not fulfilled. The issue: a database migration renamed the `order_status` column to `status` in the `orders` table. The new code references `status`, but a specific transaction path in the order processing service still references `order_status` via a raw SQL query that wasn't updated. The health check endpoint returns 200 because it doesn't query the affected table. The deployment was promoted to production because all automated checks passed. Customers are now reporting missing orders and duplicate charges.

## Interviewer Question

"Your deployment pipeline shows all green, but business transactions are failing silently. How do you detect this gap between deployment success and business success? Walk me through your investigation, fix, and prevention strategy."

## What I Should Think About

- The gap between "deployment success" and "business success" — health checks don't verify business logic
- Health check limitations: `/health` returning 200 doesn't mean the app can process orders
- Smoke tests vs end-to-end business tests: smoke tests verify the app starts, business tests verify it works
- Database migration risks: column renames, data type changes, backward compatibility
- Silent failures: no error logs, just missing data — the hardest type of failure to detect
- Reconciliation: comparing expected vs actual business outcomes (orders created vs orders processed)
- Canary analysis: business metrics during canary phase, not just error rates
- Rollback with data considerations: migration already applied, can't simply roll back code
- Monitoring business KPIs: order completion rate, payment success rate as deployment health signals
- Post-deployment verification: synthetic transactions that exercise critical business paths
- Database backward compatibility: expand-contract migration pattern
- Incident response for silent failures: how to detect, investigate, and fix without customer impact

## Ideal Answer

This is a classic "deployment succeeded but business failed" scenario. The root cause is a database migration that broke a specific code path without triggering any health check failures.

**Detection Gap**: The health check endpoint probably just verifies database connectivity (`SELECT 1`), not business logic. A proper health check would verify the application can actually process orders — query the affected table, verify the column exists, and check that the data is accessible.

**Immediate Investigation**: Check business metrics — order completion rate dropped from 99.5% to 45% after deployment. Compare order creation count vs order processing count. Check for database query errors in application logs — the raw SQL query referencing `order_status` should be throwing errors, but they might be caught and logged at a low level.

**Root Cause**: The database migration renamed `order_status` to `status` but a raw SQL query in the order processing service still references `order_status`. This query is only executed during order fulfillment, not during health checks or smoke tests.

**Immediate Fix**: Deploy a hotfix that updates the raw SQL query to reference `status` instead of `order_status`. For data integrity, implement reconciliation to identify orders that were created but not processed, and reprocess them.

**Prevention Strategy**: Implement expand-contract migration pattern: first add the new column (`status`), deploy code that reads from both columns, then remove the old column (`order_status`). This ensures zero-downtime migrations. Add business transaction health checks that verify critical paths can execute. Implement post-deployment verification with synthetic transactions that exercise order creation and processing.

## Architecture

```
Deployment Success vs Business Success Gap
============================================

Deployment Pipeline (All Green ✓)
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   Build     │  │   Test      │  │   Deploy    │
│  ✓ Pass     │  │  ✓ Pass     │  │  ✓ Rollout  │
└─────────────┘  └─────────────┘  └─────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────┐
│  Application Layer: Health Check → 200 OK ✓          │
│  Business Logic ✗: Order processing FAILS (column   │
│  not found). Order creation works. Payment works     │
│  but not linked to order.                           │
└─────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────┐
│  Database: Migration Applied ✓ (order_status → status)│
│  Health check: SELECT 1 → ✓                         │
│  Business query: WHERE order_status = 'pending' → ✗ │
└─────────────────────────────────────────────────────┘

Detection Layers:
  Layer 1: Health Checks (Failed to detect)
  Layer 2: Smoke Tests (Failed to detect)
  Layer 3: Synthetic Transactions (Would detect)
  Layer 4: Business Metrics Monitoring (Would detect)
  Layer 5: Reconciliation Jobs (Would detect)
```

## Investigation

1. **Check Business Metrics**: Compare order creation rate vs order processing rate — if they diverge after deployment, business logic is broken
2. **Review Deployment Timeline**: When was the migration applied? When was the new code deployed? Correlate with metric drops
3. **Check Application Logs**: Look for database query errors, especially "column not found" errors in the order processing service
4. **Query the Database**: Check if the migration was applied correctly — `SELECT column_name FROM information_schema.columns WHERE table_name = 'orders'`
5. **Review Migration Files**: Compare the migration SQL with the application code — identify the column name mismatch
6. **Trace the Code Path**: Follow the order processing flow from creation to fulfillment — identify where the raw SQL query is
7. **Check for Silent Failures**: Look for exception handling that catches and logs errors without failing the request
8. **Compare Expected vs Actual**: Count orders created vs orders processed — the gap represents failed transactions
9. **Identify Affected Time Window**: Determine when the issue started and how many orders are affected
10. **Assess Customer Impact**: How many customers were affected? Were payments charged without fulfillment?

## Commands

```bash
# Check order creation vs processing rates (Prometheus)
# Create instant vector showing divergence
curl -G 'http://prometheus:9090/api/v1/query' \
  --data-urlencode 'query=rate(orders_created_total[5m])' \
  --data-urlencode 'query=rate(orders_processed_total[5m])'

# Check database column existence
kubectl exec -it postgres-0 -- psql -d shopdb -c \
  "SELECT column_name FROM information_schema.columns 
   WHERE table_name = 'orders' AND column_name IN ('status', 'order_status');"

# Check for database query errors in application logs
kubectl logs -l app=order-processor -n production --tail=1000 | \
  grep -i "column.*does not exist\|undefined column\|order_status"

# Check order processing failures
kubectl exec -it postgres-0 -- psql -d shopdb -c \
  "SELECT COUNT(*) as total_orders, 
          COUNT(CASE WHEN processed_at IS NOT NULL THEN 1 END) as processed,
          COUNT(CASE WHEN processed_at IS NULL THEN 1 END) as unprocessed
   FROM orders 
   WHERE created_at > NOW() - INTERVAL '1 hour';"

# Review the problematic migration
cat migrations/20240115_rename_order_status.sql
# Should show: ALTER TABLE orders RENAME COLUMN order_status TO status;

# Check for raw SQL queries referencing old column name
grep -r "order_status" --include="*.py" --include="*.java" --include="*.go" app/

# Deploy hotfix for the raw SQL query
# Fix: UPDATE queries/order_processor.py
# Change: WHERE order_status = 'pending'
# To:     WHERE status = 'pending'

# Create reconciliation job to reprocess failed orders
cat > reconciliation-job.yaml <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: order-reconciliation
  namespace: production
spec:
  template:
    spec:
      containers:
        - name: reconcile
          image: myorg/order-reconciler:v1.0
          env:
            - name: AFFECTED_WINDOW_START
              value: "2024-01-15T14:00:00Z"
            - name: AFFECTED_WINDOW_END
              value: "2024-01-15T16:00:00Z"
      restartPolicy: Never
EOF

# Run reconciliation
kubectl apply -f reconciliation-job.yaml

# Verify reconciliation results
kubectl logs job/order-reconciliation -n production

# Post-deployment verification - synthetic transaction
cat > synthetic-transaction.yaml <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: synthetic-transactions
  namespace: production
data:
  order-flow.sh: |
    # Create test order
    ORDER_ID=$(curl -s -X POST https://api.internal.com/orders \
      -H "Content-Type: application/json" \
      -d '{"product_id": "test-123", "quantity": 1}' | jq -r '.order_id')
    
    # Wait for processing
    sleep 5
    
    # Verify order was processed
    STATUS=$(curl -s https://api.internal.com/orders/$ORDER_ID | jq -r '.status')
    if [ "$STATUS" != "processed" ]; then
      echo "CRITICAL: Order $ORDER_ID not processed (status: $STATUS)"
      exit 1
    fi
    echo "OK: Order $ORDER_ID processed successfully"
EOF
```

## Root Cause

1. **Migration Without Code Coordination**: Database migration renamed a column but not all code references were updated
2. **Raw SQL Queries**: Application uses raw SQL queries that bypass ORM column name management
3. **Insufficient Health Checks**: Health check only verifies connectivity, not business logic functionality
4. **Missing Smoke Tests**: Smoke tests don't exercise critical business transaction paths
5. **No Business Metrics Monitoring**: Order completion rate not monitored as a deployment health signal
6. **Silent Failure Handling**: Exception handling catches errors and logs them without failing the request
7. **No Post-Deployment Verification**: No synthetic transactions to verify business flows after deployment

## Immediate Mitigation

1. **Deploy Hotfix**: Update the raw SQL query to reference the new column name (`status` instead of `order_status`)
2. **Run Reconciliation**: Execute reconciliation job to identify and reprocess orders that were created but not processed
3. **Refund Affected Payments**: Process refunds for payments that were charged but orders not fulfilled
4. **Notify Customers**: Communicate with affected customers about the issue and resolution
5. **Monitor Recovery**: Track order completion rate to verify the fix is working
6. **Post-Mortem**: Conduct blameless post-mortem to identify process improvements

## Permanent Fix

1. **Expand-Contract Migration Pattern**: First add new column → deploy code reading both → remove old column. Never rename in a single step
2. **Business Transaction Health Checks**: Health check endpoints that verify critical business paths can execute
3. **Synthetic Transactions**: Automated post-deployment verification that exercises order creation and processing
4. **Business Metrics Monitoring**: Real-time monitoring of order completion rate, payment success rate as deployment health signals
5. **Reconciliation Jobs**: Automated daily reconciliation comparing expected vs actual business outcomes
6. **Code Review for Migrations**: Database migrations reviewed by both DBA and application developers
7. **Integration Tests**: End-to-end tests that verify the full order lifecycle, not just individual endpoints

## Monitoring

- **Business KPIs**: Order completion rate, payment success rate, order processing latency — monitored in real-time
- **Deployment Health**: Compare business metrics before and after deployment — alert on significant drops
- **Reconciliation Metrics**: Track daily reconciliation job results — orders created vs processed vs failed
- **Error Rates**: Monitor application logs for database query errors, especially after migrations
- **Synthetic Transaction Success Rate**: Track success rate of automated business flow verification
- **Customer Impact**: Monitor support tickets, refund requests, and customer complaints
- **Migration Status**: Track which migrations have been applied, verify column names match code expectations

## Security

- **Data Integrity**: Silent failures can cause financial discrepancies — reconciliation must verify payment and order data consistency
- **Audit Trail**: All database migrations must be logged with timestamps and who approved them
- **Access Control**: Only authorized personnel can run database migrations in production
- **Compliance**: Payment processing failures may violate PCI DSS requirements — immediate incident response required
- **Data Recovery**: Reconciliation jobs must handle partial failures and resume without data loss
- **Incident Response**: Silent failures require different response protocols than obvious outages

## Production Considerations

- **HA**: Silent failures don't cause visible outages but can cause data corruption — equally dangerous
- **Scalability**: Reconciliation jobs must handle large volumes of affected orders without impacting production
- **Reliability**: Expand-contract migrations ensure zero-downtime schema changes
- **Cost**: Reconciliation and refunds have real financial cost — prevention is cheaper than remediation
- **Compliance**: Financial transaction failures may require regulatory reporting
- **Operational**: Runbooks for silent failure scenarios, automated reconciliation, and customer communication
- **Migration Strategy**: Every database migration must follow expand-contract pattern for zero-downtime deployments
- **Testing**: End-to-end tests that verify business transactions, not just API responses

## Senior-Level Answer

I'd detect this gap through business metrics monitoring — tracking order completion rate as a deployment health signal. The fix involves a hotfix to the raw SQL query, reconciliation to reprocess failed orders, and refunds for affected customers. Long-term, I'd implement expand-contract migrations (add new column → deploy code → remove old column), business transaction health checks, and post-deployment synthetic transactions. The key insight is that health checks verify the application is running, not that it's working — you need business-level verification to catch silent failures.

## Architect-Level Answer

This scenario reveals a critical gap between deployment verification and business verification. I'd architect a multi-layer detection system: Layer 1 — health checks that verify business logic, not just connectivity; Layer 2 — synthetic transactions that exercise critical business flows post-deployment; Layer 3 — real-time business metrics monitoring with automated rollback triggers; Layer 4 — daily reconciliation comparing expected vs actual business outcomes. The strategic fix is the expand-contract migration pattern: never rename columns in a single migration, always use additive changes that maintain backward compatibility. This requires organizational process change — database migrations must be reviewed by both DBA and application teams, and every migration must have a rollback plan that accounts for data state.

## Expand-Contract Migration Pattern

```
Expand-Contract Pattern for Column Rename
==========================================

Step 1: EXPAND — Add new column (backward compatible)
  ALTER TABLE orders ADD COLUMN status VARCHAR(50);
  Both columns exist: order_status ✓, status (null initially) ✓

Step 2: MIGRATE — Copy data and keep in sync
  UPDATE orders SET status = order_status;
  CREATE TRIGGER sync_status BEFORE UPDATE ON orders
    SET NEW.status = NEW.order_status IF NEW.status IS NULL;
  Both columns always in sync: old code ✓, new code ✓

Step 3: DEPLOY — Update all code to use new column
  All services updated to use 'status'. No code references order_status.

Step 4: CONTRACT — Remove old column (after verification)
  ALTER TABLE orders DROP COLUMN order_status; DROP TRIGGER sync_status;
  Zero-downtime migration complete.
```

## Business Transaction Health Check Design

The `/health/ready` endpoint should verify (not just `SELECT 1`):
- Database connectivity: `SELECT 1`
- Required columns exist: `SELECT column_name FROM information_schema.columns WHERE table_name = 'orders' AND column_name = 'status'`
- Required services reachable: Check payment service, inventory service
- Business logic functional: Can we create and process a test order?

```python
@app.route('/health/ready')
def readiness():
    checks = {}
    # Database connectivity
    try:
        db.execute("SELECT 1")
        checks['database'] = 'ok'
    except Exception as e:
        checks['database'] = f'error: {str(e)}'
    # Verify required columns exist
    try:
        result = db.execute(
            "SELECT column_name FROM information_schema.columns "
            "WHERE table_name = 'orders' AND column_name = 'status'"
        )
        checks['schema'] = 'ok' if result.fetchone() else 'error: status column missing'
    except Exception as e:
        checks['schema'] = f'error: {str(e)}'
    # Business logic verification
    try:
        test_id = db.execute(
            "INSERT INTO orders (status, created_at) VALUES ('test_health_check', NOW()) RETURNING id"
        ).fetchone()[0]
        db.execute("DELETE FROM orders WHERE id = %s", (test_id,))
        checks['business_logic'] = 'ok'
    except Exception as e:
        checks['business_logic'] = f'error: {str(e)}'
    all_ok = all(v == 'ok' for v in checks.values())
    return jsonify({'status': 'ready' if all_ok else 'not_ready', 'checks': checks}), 200 if all_ok else 503
```

## Incident Response for Silent Failures

```
Silent Failure Response Protocol
==================================

Detection Sources:
1. Business metrics monitoring (order completion rate drop)
2. Reconciliation job (orders created vs processed mismatch)
3. Customer complaints (missing orders, duplicate charges)
4. Synthetic transaction failures

Response Steps:
1. CONFIRM — Verify the failure is real, not a monitoring artifact
2. CONTAIN — Prevent further damage (disable affected code path, rollback)
3. ASSESS — Determine scope: how many transactions affected, time window
4. COMMUNICATE — Notify stakeholders, update status page
5. REMEDIATE — Fix the root cause, deploy hotfix
6. RECONCILE — Reprocess failed transactions, refund affected payments
7. VERIFY — Run synthetic transactions to confirm fix works
8. DOCUMENT — Post-mortem, process improvements, monitoring additions
```

## Follow-Up Questions

1. How do you implement the expand-contract migration pattern for a column rename without any downtime?
2. How do you design health checks that verify business logic without creating performance overhead?
3. How do you handle reconciliation when the affected time window is hours and millions of transactions are involved?
4. How do you prevent database migrations from breaking application code in a microservices architecture?
5. What's the difference between a silent failure and an obvious outage, and how does your incident response differ?
